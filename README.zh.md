# seo-aeo-bootstrap

一个 Claude Code skill：把网站从"搜索引擎和 AI 都看不见"做到"被抓取、被引用、可归因、效果可验证"，一次搭好 SEO 和 AEO/GEO（AI 答案引擎优化）。

方法论先在 [Iriseed](https://iriseed.com)（Next.js + Vercel + Supabase）上完整落地，又在一家创业公司的官网上复用了一遍，代码模板都来自实战。

[English](README.md)

## 最关键的一项检查

GPTBot、ClaudeBot、PerplexityBot、百度、搜狗都**不执行 JavaScript**。如果正文是客户端渲染的，它们只能看到标题和一句摘要。

Iriseed 实测：一篇源文 4,329 字的文章，爬虫只能看到 **192 字**；服务端渲染一份与交互版一字不差的语义化全文之后，变成 **4,292 字**。这个数不对，其他 AEO 动作都是白做。

## 包含什么

- **Phase 0 诊断**：量爬虫可见字数，检查 sitemap / robots / llms.txt / metadata / JSON-LD / 控制台，按"命门 → 缺失 → 可优化"排序
- **Phase 1 四件套**：sitemap、robots（显式欢迎 AI 爬虫）、llms.txt、逐页 metadata + JSON-LD，全部从同一个内容注册表生成
- **Phase 2 全文对爬虫可见**：服务端渲染 + 验证
- **Phase 3 探针**：边缘层爬虫探针（23 种爬虫）、15+ AI 引擎引流归因（ChatGPT、Perplexity、Kimi、豆包……）、站长自净
- **Phase 4 官方控制台**：GSC 和 Bing 的正确接入顺序
- **Phase 5 解读节奏**：当天、每周、每月看什么，以及零点击引用的盲区怎么补
- 内容层写作手法，以及中文生态的特别注意事项

## 安装

```bash
git clone https://github.com/witch-Judy/seo-aeo-bootstrap
cp -R seo-aeo-bootstrap/zh ~/.claude/skills/seo-aeo-bootstrap
```

## License

MIT
