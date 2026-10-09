---
name: seo-aeo-bootstrap
description: 给任何网站从零搭建完整的 SEO/AEO（AI 答案引擎优化）体系：诊断爬虫可见性、补服务端全文渲染、建 sitemap/robots/llms.txt 四件套、部署爬虫探针与 AI 引流归因、注册 Google Search Console 和 Bing Webmaster、并用可验证的方式监控效果。当用户提到 SEO、AEO、GEO、收录、被 AI 引用、"为什么 Google/ChatGPT 搜不到我的站"、Search Console、sitemap、robots.txt、llms.txt、爬虫抓取、新站上线要做优化、想看 AI 带来多少流量时使用——即使用户只提到其中一个环节，也应加载本 skill 检查整条链路。
---

# SEO/AEO 建站方法论

把一个网站从"引擎看不见"做到"被引用、可归因、效果可验证"。
方法论在 Iriseed（iriseed.com，Next.js + Vercel + Supabase）上完整落地过一次，
代码模板都来自实战。

## 心法（先读这个再动手）

**AEO 的因果链只有三环，每一环用不同性质的数据：**

| 环节 | 问题 | 数据性质 |
|---|---|---|
| ① 抓取 | AI/搜索爬虫来抓了没有 | **普查**——自己的探针，每次抓取都在案 |
| ② 展示 | 内容在 AI 答案/搜索结果里出现了没有 | 只有官方控制台可信（GSC、Bing），其余全是抽样猜测 |
| ③ 引流 | 展示之后有人点进来没有 | **普查**——referrer/UTM，每次点击都在案 |

两头（①③）是普查，自己就能拿到 100% 可信的数据；中间（②）只认官方数。
第三方"AEO 分数"产品的原理是拿自选问题集去抽样问各引擎 API——问题是他们选的、
答案非确定、API ≠ 真实用户界面、没有对照组。不要买分数，用①③两头夹逼②。
详见 `references/measurement.md`。

**两个来自实战的教训：**
- 埋点宜早不宜迟。Iris 在埋点上线前有 13 个注册用户，他们看了什么、卡在哪里，永远查不回来了。探针要在推广之前上线，而不是之后。
- 外发的每一个链接都挂 UTM。否则流量来了也认不出来源。

## Phase 0 — 诊断（10 分钟，先看病再开药）

**最关键的一项检查：不执行 JS 的爬虫能看到多少字。**
GPTBot、ClaudeBot、PerplexityBot、百度、搜狗全都不执行 JS——
如果站是 SPA/客户端渲染，它们看到的可能只有标题和一句摘要。

```bash
curl -s https://目标站/某内容页 | python3 -c "
import sys, re
html = sys.stdin.read()
text = re.sub(r'<script[\s\S]*?</script>', '', html)
text = re.sub(r'<[^>]+>', ' ', text)
text = re.sub(r'\s+', ' ', text)
print('HTML 总长:', len(html), '| 爬虫可见文本:', len(text))
print(text[:400])
"
```

可见文本远小于页面实际内容 → 命门在 Phase 2，其他都往后排。

其余检查（存在与否 + 质量）：
- `/sitemap.xml`（是否覆盖全部内容页、是否随内容自动更新）
- `/robots.txt`（是否误封 AI 爬虫；不少模板默认 Disallow 一切）
- `/llms.txt`（llmstxt.org 约定，给 AI 引擎的站点说明书）
- 内容页的 `<title>`/`description`/OG 标签是否逐页生成
- JSON-LD 结构化数据是否存在
- 是否已注册 GSC / Bing Webmaster

把诊断结果按"命门 → 缺失 → 可优化"排序汇报，不要平铺。

## Phase 1 — 四件套基建

sitemap、robots、llms.txt、逐页 metadata + JSON-LD。
核心原则：**全部从同一个内容注册表生成**（数据库、CMS API 或 `articles.ts` 这类
单一数据源），发新内容时四件套自动同步，永不腐烂。手写清单第二周就会过期。

robots 里显式欢迎 AI 爬虫（GPTBot/ClaudeBot/PerplexityBot/Bytespider 等，
清单见 code-patterns）；llms.txt 里写清楚"适合引用的场景"——这是在教 AI 引擎
什么时候该引用你。

代码模板（Next.js 为主，含通用原则）：`references/code-patterns.md` 第 1 节。

## Phase 2 — 命门：全文对爬虫可见

如果 Phase 0 查出正文不在服务端 HTML 里，这是优先级最高的一件事，
其他 AEO 动作在它面前都是白做——引擎引用你的前提是能读到你。

做法按代价从小到大：
1. 框架本来支持 SSR/SSG → 把内容获取从客户端 useEffect 挪到服务端/构建期即可
2. 内容在 iframe/客户端渲染壳里 → 服务端额外渲染一份**与交互版一字不差**的
   语义化全文（h1/h2/p 结构），客户端壳挂载后隐藏它；加载失败时重新亮出来当兜底
3. 纯静态托管改不了渲染 → 至少把全文塞进构建产物

**内容一致性原则**：服务端版和用户版内容必须一字不差——这是"渐进增强"，
不是 cloaking。给爬虫看不一样的内容才是 cloaking，会被惩罚。

改完必须验证：用 Phase 0 的 curl 命令量字数，和源文对照应基本相等
（Iris 实测：192 字 → 4292 字 / 源文 4329 字）。

实现模板：`references/code-patterns.md` 第 2 节。

## Phase 3 — 探针（效果监控的自建部分）

三个探针，都写进自己的事件表（普查，不依赖任何第三方）：

1. **爬虫探针**：middleware/边缘层按 UA 识别已知爬虫，异步记录
   `bot_hit`（爬虫名、路径、国家）。要点：不阻塞响应、失败静默、
   爬虫请求跳过 session 逻辑。爬虫不执行 JS，前端埋点对它们完全失明，
   所以这一层必须在服务端/边缘做。
2. **AI 引流归因**：前端埋点采集 referrer + UTM，按引擎清单分类
   （chatgpt.com、perplexity.ai、kimi、豆包等 15+）。
   ChatGPT 外链自带 `utm_source=chatgpt.com`。
3. **站长自净**：统计面板默认剔除站长本人（登录 user_id + 关联匿名 id +
   本地开发流量）。站长自己的访问会把小站的数据完全带偏。

上线后立即验证探针：伪装爬虫 UA curl 一次，确认落库，再删测试行。
实现模板与 bot/引擎清单：`references/code-patterns.md` 第 3 节。

## Phase 4 — 官方控制台

1. **代码里预留验证位**：metadata 读 `GOOGLE_SITE_VERIFICATION` /
   `BING_SITE_VERIFICATION` 环境变量（模板见 code-patterns 第 4 节）。
   以后换站/换号只改环境变量，不改代码。
2. **GSC**：用户去 search.google.com/search-console 建"网址前缀"资源 →
   选"HTML 标记"拿 token → 你存进部署平台环境变量 → 触发重新部署 →
   **确认线上 meta 已渲染**（curl grep）→ 让用户点验证 → 提交 sitemap.xml。
   顺序错了必失败：先部署后验证。
3. **Bing Webmaster**：优先"从 GSC 导入"（一键）。GSC 资源太新时导入会报
   "unable to fetch data"——正常，稍后重试或手动加站 + meta 验证。
   Bing 的 **AI Performance (BETA)** 报告 = Copilot/Bing AI 面的官方引用数据，
   这是目前唯一一块官方 AEO 瓷砖，但它只代表微软，不代表所有 AI。

## Phase 5 — 解读与节奏

- 部署当天：`/stats` 类面板即时可看爬虫动静；GSC/Bing 数据 1-2 天后开始累积
- **第一个值得看的信号**：爬虫抓的是哪几篇——那是 AI 引擎认为你站里最值得引用的内容
- 周检查：爬虫抓取趋势、AI 引流有无破零、GSC 曝光词（用户实际怎么搜到你）
- 盲区：零点击引用（AI 说了你的内容但用户没点）。补法是 DIY 抽样探测——
  拿每篇内容回答的真实问题，固定问题集、固定频率问各引擎，记录被引用与否。
  设计要点见 `references/measurement.md`。先让爬虫探针跑两周再上这个。

## 内容层：让内容值得被引用

基建决定"能不能被读到"，内容写法决定"会不会被引用"。
写作/改稿时的具体手法（答案前置、自包含段落、问题式标题、实体具体化、
独家数据、E-E-A-T 信号等）：`references/content-tips.md`。
给内容站做优化时，这份和基建同等重要。

## 完工验收清单

```bash
# 1. 爬虫可见全文（数字应接近源文）
curl -s https://站/内容页 | python3 -c "…（Phase 0 脚本）"
# 2. 四件套在线
curl -sI https://站/sitemap.xml | head -1
curl -s https://站/robots.txt | head -5
curl -s https://站/llms.txt | head -5
# 3. 探针活着（伪装 UA 后查库，测完删行）
curl -s -A "GPTBot/1.2" https://站/内容页 -o /dev/null
# 4. 验证标签已渲染
curl -s https://站/ | grep -o 'google-site-verification[^>]*'
```

全绿 + GSC/Bing 已提交 sitemap = 交付。之后的事是等数据、看 Phase 5。
