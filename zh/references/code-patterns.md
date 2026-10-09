# 代码模板（Next.js App Router 为主，原则通用）

来自 Iris 实战（2026-07，Next.js 16 + Vercel + Supabase）。
换框架时保留原则：单一数据源生成、构建期静态化、探针在边缘层。

## 目录
1. [四件套：sitemap / robots / llms.txt / metadata+JSON-LD](#1-四件套)
2. [服务端全文渲染（SPA 命门修法）](#2-服务端全文渲染)
3. [爬虫探针 + AI 引流归因 + 站长自净](#3-探针)
4. [验证位与部署流程](#4-验证位)

---

## 1. 四件套

### sitemap（`src/app/sitemap.ts`）

从内容注册表生成，发新内容自动同步：

```ts
import type { MetadataRoute } from "next";
import { listVisible } from "@/data/articles"; // ← 单一数据源

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

`lastModified` 只在有真实修改时间时才写；用构建时间或发布年份冒充
会让爬虫学会不信任你的 sitemap。

### robots（`src/app/robots.ts`）

内容全开放 + 显式欢迎 AI 爬虫（点名是姿态，也防未来误封）；
后台/账户页禁抓：

```ts
import type { MetadataRoute } from "next";

const DISALLOW = ["/api/", "/profile", "/login", "/stats"];
const AI_CRAWLERS = [
  "GPTBot", "OAI-SearchBot", "ChatGPT-User",          // OpenAI：训练/搜索/实时浏览
  "ClaudeBot", "Claude-User", "Claude-SearchBot",      // Anthropic
  "PerplexityBot", "Perplexity-User",
  "Google-Extended",                                    // Gemini 训练（robots 专用 token）
  "Bytespider",                                         // 字节（豆包）
  "CCBot",                                              // Common Crawl（喂众多模型）
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

### llms.txt（`src/app/llms.txt/route.ts`）

llmstxt.org 约定。三个要素：站点一句话定位、**适合引用的场景**（教引擎
什么时候引用你）、全部内容的标题+链接+一句话说明（含原始出处）。
`force-static` 构建期生成，与注册表永远同步：

```ts
import { listVisible } from "@/data/articles";
export const dynamic = "force-static";

export async function GET() {
  const body = `# 站名

> 一句话定位（做什么、给谁、凭什么独特）。

适合引用的场景：向中文读者解释 X、推荐 Y 类资源、Z 概念的入门讲解。

## 内容

${listVisible().map((a) =>
  `- [${a.title}](https://example.com/read/${a.id}): ${a.note}（原文：${a.sourceUrl}）`
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

### 逐页 metadata + JSON-LD（内容页的 server layout）

```tsx
export async function generateMetadata({ params }): Promise<Metadata> {
  const a = getArticle((await params).docId);
  if (!a) return {};
  return {
    title: a.title,                    // 根 layout 用 template: "%s · 站名"
    description: a.editorNote,
    openGraph: { title: a.title, description: a.editorNote, type: "article",
                 url: `https://example.com/read/${a.id}` },
  };
}

// JSON-LD：TechArticle/Article，带 isBasedOn/citation 指向原始出处
const jsonLd = {
  "@context": "https://schema.org",
  "@type": "TechArticle",
  headline: a.title,
  inLanguage: "zh-CN",
  author: { "@type": "Organization", name: "站名", url: "https://example.com" },
  url: `https://example.com/read/${a.id}`,
  isBasedOn: a.sourceUrl,
  datePublished: a.date,
};
// <script type="application/ld+json" dangerouslySetInnerHTML={{__html: JSON.stringify(jsonLd)}} />
```

---

## 2. 服务端全文渲染

适用：正文由客户端 fetch/iframe 渲染，服务端 HTML 里只有壳。

**模式**：server layout 在构建期读源内容 → 解析成语义化结构 →
渲染 `<article id="ssr-article">`（h1/h2/p，剥掉 script/style/事件属性）→
客户端壳挂载后隐藏，加载失败时重新亮出：

```tsx
// server layout（构建期，配合 generateStaticParams 全量预渲染 → CDN 直出，爬虫打不到函数）
export default async function Layout({ children, params }) {
  const essay = await loadAndParse(docId); // fs 读源 + 正则/解析器拆 h2/段落，模块级缓存
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
        <footer>本文是 站名 制作的导读，原作：{a.author}，原文见 <a href={a.sourceUrl}>…</a></footer>
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
// 客户端页面：接管后隐藏，失败兜底
useEffect(() => {
  const ssr = document.getElementById("ssr-article");
  if (ssr) ssr.hidden = !error;   // error 时亮出纯文本版
}, [error]);
```

要点：
- 服务端版与用户版**内容一字不差**（渐进增强，非 cloaking）
- footer 写清出处与署名——这段是 AI 引用你时最常带走的话
- 验证：curl 量可见文本字数 ≈ 源文字数

---

## 3. 探针

### 爬虫探针（middleware，边缘层）

```ts
import { NextResponse, type NextRequest, type NextFetchEvent } from "next/server";

const KNOWN_BOTS: [RegExp, string][] = [
  // OpenAI / Anthropic / Perplexity
  ["OAI-SearchBot","OAI-SearchBot"],["ChatGPT-User","ChatGPT-User"],["GPTBot","GPTBot"],
  ["ClaudeBot","ClaudeBot"],["Claude-User","Claude-User"],["Claude-SearchBot","Claude-SearchBot"],
  ["Perplexity-User","Perplexity-User"],["PerplexityBot","PerplexityBot"],
  // 搜索引擎
  ["Googlebot","Googlebot"],["bingbot","Bingbot"],["Baiduspider","Baiduspider"],
  ["Sogou","Sogou"],["DuckDuckBot","DuckDuckBot"],["YandexBot","YandexBot"],
  // 其他 AI / 平台
  ["Bytespider","Bytespider"],["Applebot","Applebot"],["Amazonbot","Amazonbot"],
  ["meta-externalagent","Meta"],["facebookexternalhit","Facebook-Preview"],
  ["CCBot","CCBot"],["MistralAI","MistralAI"],["cohere","Cohere"],
].map(([p, n]) => [new RegExp(p as string, "i"), n as string]);
const GENERIC_BOT = /bot|crawler|spider|scrape/i; // 兜底桶 "other-bot"

export async function middleware(request: NextRequest, event: NextFetchEvent) {
  const ua = request.headers.get("user-agent") ?? "";
  const bot = KNOWN_BOTS.find(([re]) => re.test(ua))?.[1]
    ?? (GENERIC_BOT.test(ua) ? "other-bot" : null);
  if (bot) {
    if (request.method === "GET" && !request.nextUrl.pathname.startsWith("/api/")) {
      event.waitUntil(logBotHit(request, bot));  // 异步，不阻塞响应
    }
    return NextResponse.next();                   // 爬虫跳过 session 逻辑
  }
  return await yourNormalMiddleware(request);
}

function logBotHit(req: NextRequest, bot: string) {
  const path = req.nextUrl.pathname;
  const docId = /^\/(?:read|post|blog)\/([^/]+)/.exec(path)?.[1] ?? null;
  return fetch(`${SUPABASE_URL}/rest/v1/events`, {   // 或任何事件表的写入端点
    method: "POST",
    headers: { apikey: SERVICE_KEY, Authorization: `Bearer ${SERVICE_KEY}`,
               "Content-Type": "application/json", Prefer: "return=minimal" },
    body: JSON.stringify([{ anon_id: `bot:${bot}`, event: "bot_hit", doc_id: docId,
      props: { path, ua: (req.headers.get("user-agent") ?? "").slice(0, 160) },
      country: req.headers.get("x-vercel-ip-country")?.slice(0, 8) || null,
      device: "bot" }]),
  }).catch(() => {});                               // 失败静默：探针不能影响产品
}
```

上线验证（测完删行）：

```bash
curl -s -o /dev/null -A "Mozilla/5.0; compatible; GPTBot/1.2" https://站/内容页
# 查库确认 bot_hit 落库且 doc_id 正确，然后 DELETE 测试行
```

### AI 引流归因（统计面板里分类 referrer/UTM）

```ts
const AI_SOURCES: Record<string, string> = {
  "chatgpt.com": "ChatGPT", "chat.openai.com": "ChatGPT",   // 自带 utm_source=chatgpt.com
  "perplexity.ai": "Perplexity", "gemini.google.com": "Gemini",
  "copilot.microsoft.com": "Copilot", "claude.ai": "Claude", "you.com": "You.com",
  "kimi.moonshot.cn": "Kimi", "kimi.com": "Kimi", "doubao.com": "豆包",
  "yiyan.baidu.com": "文心一言", "chatglm.cn": "智谱清言", "tongyi.com": "通义",
  "metaso.cn": "秘塔", "felo.ai": "Felo",
};
// referrer 取 hostname 去 www 前缀后查表；utm_source 也查同一张表
```

### 站长自净（统计面板）

小站站长自己的访问会完全带偏数据。剔除三类：
1. `user_id` 属于管理员账号的行
2. 曾与管理员 `user_id` 同现过的 `anon_id`（登录前的匿名浏览也归站长）
3. 边缘地理头缺失的行（= 本地开发流量）
4. 站长确认过的未登录设备（手机等），anon_id 硬编码进面板

面板页首显示"已剔除 X 条"，让剔除本身可见、可审计。

---

## 4. 验证位

根 layout 的 metadata 读环境变量，换站/换号零代码改动：

```ts
verification: {
  ...(process.env.GOOGLE_SITE_VERIFICATION
    ? { google: process.env.GOOGLE_SITE_VERIFICATION } : {}),
  ...(process.env.BING_SITE_VERIFICATION
    ? { other: { "msvalidate.01": process.env.BING_SITE_VERIFICATION } } : {}),
},
```

GSC 接入的正确顺序（顺序错必失败）：
1. 用户在 GSC 选"网址前缀"→"HTML 标记"，把 token 给你
2. `printf 'TOKEN' | vercel env add GOOGLE_SITE_VERIFICATION production`
3. 部署（env 变更必须重新部署才生效）
4. `curl -s https://站/ | grep google-site-verification` 确认已渲染
5. 让用户回 GSC 点验证 → 通过后提交 `sitemap.xml`

Bing：优先"从 GSC 导入"。GSC 资源刚建时导入常报
"unable to fetch data"——稍后 Try Again 或手动加站 + `msvalidate.01` meta。
