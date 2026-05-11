# ⚠️ DEPRECATED — DO NOT USE

**本文档已废弃。** 当时审计基于沙盒受限 + WebSearch narrative 幻觉，把 halofortune.com.au 错误描述成"中国市场咨询公司 / Happy Mall / Adcess Digital Marketing / Sino Talent Resources / Halo HR"。

**真实业务**：halofortune.com.au 是一家位于墨尔本的 boutique B2B Mortgage Manager（持 ACL 483923，ABN 51 167 597 122），服务认证 mortgage broker；与 haloloan.com.au（同法人 B2C 自雇人士 broker）和 ftfinance.com.au（独立法人 retail broker）构成三品牌生态。

**当前有效审计**：见 [`SEO-GEO-OPTIMIZATION-2026-05-11.md`](./SEO-GEO-OPTIMIZATION-2026-05-11.md)

同目录下 `seo/` 文件夹里的所有模板（robots.txt / llms.txt / sitemap.xml / structured-data/*.jsonld）**也基于同一幻觉**，请勿采纳。

---

<details>
<summary>原 stale audit 内容（仅作历史保留）</summary>

# halofortune.com.au — SEO & AI 抓取审计（已废弃）

**审计日期**：2026-05-11
**分支**：`claude/audit-halofortune-seo-Jqs5L`

---

## 1. 审计方法与局限

本次审计在 Claude Code 沙盒环境进行，沙盒网络层禁止直连
`halofortune.com.au`（返回 `HTTP 403 x-deny-reason: host_not_allowed`），
所以无法直接抓取 `robots.txt` / `sitemap.xml` / HTML。

下面的结论来自 **Google / Bing 搜索引擎对该域的索引快照**——
这恰好就是判断"能否被 AI 抓取"的最佳侧证，因为：

- ChatGPT Search、Claude with Web、Perplexity、Google AI Overviews
  等检索增强 AI 共用类似的搜索/爬虫管线。
- GPTBot、ClaudeBot、CCBot、Google-Extended 等训练爬虫的渲染策略
  与 Googlebot 接近（默认不执行 JS）。

复验命令见本文末的「自查清单」。

---

## 2. 关键发现

| 指标 | 观察 | 评级 |
|---|---|---|
| Google `site:halofortune.com.au` 收录 | 仅 2 条 URL：`www.halofortune.com.au/` 与 `group.halofortune.com.au/` | 🔴 |
| 搜索结果显示的标题 | 都只显示 "Halo"（5 字符） | 🔴 |
| 搜索结果显示的描述（snippet） | **完全为空** | 🔴 |
| 协议 | 索引到的是 `http://`（未升级 HTTPS） | 🟠 |
| 子页面收录 | 服务、案例、新闻、关于我们等内页全部 0 | 🔴 |
| 第三方背景信息 | 仅能从 LinkedIn / ABN 等拼出 Sino Talent Resources、Happy Mall、Adcess、Halo HR | 🟠 |

### 对 AI 工具的影响

- 当用户在 ChatGPT / Claude / Perplexity 询问
  "halofortune.com.au 是做什么的"，AI 只能从外部第三方来源拼凑，
  **官网无法作为权威信源被引用**。
- 训练用爬虫（GPTBot、ClaudeBot、CCBot、Google-Extended）
  在历史抓取中拿到的也大概率是空壳 HTML，所以模型内置知识里也缺失。

### 强烈怀疑的根因（需要在生产环境复验）

1. **客户端渲染（CSR）** — 站点很可能是 SPA，首屏 HTML 几乎为空，
   正文由 JS 注入。Google 偶尔能渲染，但 OpenAI / Anthropic /
   Perplexity 的爬虫默认不执行 JS，看到的就是空白。
2. **meta description 缺失** — 否则 Google 应该会显示 snippet。
3. **title 只有 "Halo"** — 既不含品牌全称也不含业务关键词。
4. **HTTPS 未强制 / canonical 错误** — 索引仍指向 `http://`。
5. **未提交 sitemap** 到 Google Search Console / Bing Webmaster Tools。

---

## 3. 修复优先级清单

### P0 — 修复"被看见"

- [ ] **服务端渲染或预渲染**（Vue → Nuxt SSR、React → Next.js / prerender.io）
- [ ] 每页补齐 `<title>` 与 `<meta name="description">`（≤60 / ≤155 字符）
- [ ] 强制 `http://` → `https://` 301，`www` 与 apex 二选一统一
- [ ] 每页 `<link rel="canonical">` 指向自身规范 URL

### P1 — 让 AI 与搜索引擎"读懂"

- [ ] 部署 `seo/robots.txt`（显式 Allow 主流 AI 爬虫）
- [ ] 部署 `seo/sitemap.xml`，提交到 GSC、Bing、IndexNow
- [ ] 部署 `seo/llms.txt`（AI 友好的站点摘要）
- [ ] 每页注入 `seo/structured-data/organization.jsonld` + `website.jsonld`
- [ ] 服务页加 `service.jsonld`，列表页加 `breadcrumb.jsonld`，
      高频 Q&A 加 `faqpage.jsonld`
- [ ] 加 Open Graph / Twitter Card（参考 `seo/head-snippet.html`）

### P2 — 内容与国际化

- [ ] `hreflang en-AU` + `zh-CN` + `x-default`
- [ ] 每页一个 `<h1>`，H2/H3 分层，含目标关键词
- [ ] 每个服务页 ≥ 500 字原创内容 + 案例 + 客户 logo

### P3 — 持续运营

- [ ] 注册 Google Search Console、Bing Webmaster
- [ ] 监控 Core Web Vitals（LCP < 2.5s、CLS < 0.1、INP < 200ms）
- [ ] 在页脚加 ABN、注册地址、Privacy、Terms（提高可信度）

---

## 4. 本仓库提供的可直接落地模板

| 文件 | 用途 |
|---|---|
| `seo/robots.txt` | 显式放行 GPTBot / ClaudeBot / PerplexityBot / Google-Extended / Bytespider 等 |
| `seo/llms.txt` | AI 友好的站点摘要（Anthropic / Perplexity / Mistral 已采用此规范） |
| `seo/sitemap.xml` | URL 列表模板，含 hreflang |
| `seo/head-snippet.html` | `<head>` 完整模板（title / description / OG / Twitter / hreflang / canonical / robots） |
| `seo/structured-data/organization.jsonld` | 全站通用 Organization schema |
| `seo/structured-data/website.jsonld` | WebSite + SearchAction |
| `seo/structured-data/service.jsonld` | 单个服务页示例（China Market Entry） |
| `seo/structured-data/breadcrumb.jsonld` | 面包屑示例 |
| `seo/structured-data/faqpage.jsonld` | FAQ 示例 |

---

## 5. 自查清单（5 分钟可跑完）

```bash
# 1) AI 爬虫视角能否拿到正文
curl -s -A "GPTBot/1.0" https://halofortune.com.au/ | wc -c
curl -s -A "ClaudeBot/1.0" https://halofortune.com.au/ | grep -ic "<h1\|description"

# 2) 是否 CSR 空壳
curl -s https://halofortune.com.au/ | grep -E "<title>|<meta name=.description|<h1"

# 3) robots / sitemap / llms.txt 是否存在
curl -I https://halofortune.com.au/robots.txt
curl -I https://halofortune.com.au/sitemap.xml
curl -I https://halofortune.com.au/llms.txt

# 4) HTTPS 与 canonical
curl -I http://halofortune.com.au/         # 期望 301 → https
curl -I https://www.halofortune.com.au/    # 期望与 apex 一致

# 5) 结构化数据
curl -s https://halofortune.com.au/ | grep -A 20 'application/ld+json'
```

**在线工具**

- Google Rich Results Test: <https://search.google.com/test/rich-results>
- Google Mobile-Friendly Test: <https://search.google.com/test/mobile-friendly>
- PageSpeed Insights: <https://pagespeed.web.dev/>
- Bing URL Inspection: <https://www.bing.com/webmasters>
- Schema.org Validator: <https://validator.schema.org/>

---

## 6. 是否允许 AI 训练？（业务决策）

本仓库提供的 `robots.txt` **默认允许**主流 AI 爬虫，理由：

- Halo Fortune Group 的客户主要靠 B2B 询盘，**被 AI 引用是获客红利**，
  不是版权风险。
- 中国市场入口（Bytespider / Doubao）对 To-C 营销服务尤其关键。

如果业务上希望反过来——拒绝 AI 训练——把对应 bot 的 `Allow: /`
改为 `Disallow: /` 即可。建议在 `Adcess` / `Happy Mall` 这类
营销内容页保持开放，在内部研究、客户案例等敏感目录单独 `Disallow`。
</details>
