# Halo Fortune Group / Halo Loan / Fingertips Finance
## SEO + GEO 修改优化建议（交接文档）

**生成日期**：2026-05-11
**审计范围**：halofortune.com.au · haloloan.com.au · ftfinance.com.au
**审计方法**：直接 curl 真实生产域名 + GPTBot/ClaudeBot UA 探测 + JSON-LD 抽取 + HTML 头分析
**目标读者**：实施这些修改的下一位 Claude（或工程师）

---

## 0. 给接手 Claude 的关键说明

**你在做什么**：实施一份基于 2026-05-11 真实生产数据的 SEO/GEO 审计修复清单。每条问题都有：
- 证据（实际 curl 输出片段）
- 文件位置（参照 production 仓库 `Halo/Broker portal/v2-portal/` 结构）
- 具体代码改动方向（不是完整 diff，需你根据真实文件内容调整）
- 验证命令（实施后跑一遍确认）

**绝对不要做的事**（合规防火墙 — 错了会留法律风险）：

1. **不要让 halofortune.com.au / haloloan.com.au 在任何公开可见的地方（HTML body、llms.txt、JSON-LD `sameAs`、schema 关系、CTA、footer、blog 内容）出现 `ftfinance` / `Fingertips Finance` / `掌忠贷` 任一字符串。**
2. **不要让 ftfinance.com.au 在任何公开可见的地方出现 `halofortune` / `haloloan` / `Halo Fortune` / `Halo Loan`。**
3. halofortune 与 haloloan **可以互相公开提及**（它们是同一法人 Halo Fortune Group Pty Ltd 的两个面）。当前 halofortune 的 llms.txt 末尾"Related brand"段、haloloan 的 llms.txt 第二段提及 halofortune URL，**保持现状**。
4. 这三个域名的法人、牌照、AFCA / FBAA / MFAA 会员号**不能混用**。下面 §1 列出三者各自的合规身份，每个修改都要核对身份归属正确。

**还有几件事不要做**：

- 不要重做已经部署正常的基础设施。所有三域的 `robots.txt`、`llms.txt`、`sitemap.xml`、`.well-known/ai.txt`、SSR、HTTPS 308 都已上线。这份文档列出的是**已部署内容里的具体 gap**，不是"从零搭一套"。
- 不要采纳之前 PR #1 / PR #2 关于"halofortune 是中国市场咨询公司 / Happy Mall / Adcess Digital Marketing / Sino Talent Resources"的描述。**那些是基于幻觉的 stale audit**。halofortune 是 boutique B2B Mortgage Manager，不要往那个方向写任何文案或 schema。

---

## 1. 业务与法律实体上下文

| 域 | 品牌 | 业务定位 | 法人 | 牌照 |
|---|---|---|---|---|
| **halofortune.com.au** | Halo Fortune | **B2B** Mortgage Manager，服务认证 broker（< 100 席位）通过 Halo Loan wholesale 产品线 + Halo Copilot 工具 | Halo Fortune Group Pty Ltd | **ACL 483923**, ABN 51 167 597 122, ACN 167 597 122, **MFAA 660806**, **AFCA 45759** |
| **haloloan.com.au** | Halo Loan 环澳信贷 | **B2C** retail broker，服务澳洲自雇人士（ABN 持有人 / sole trader / contractor / hospitality / trade / IT），中英双语，全程线上 | **同上** Halo Fortune Group Pty Ltd | **同上** ACL 483923（覆盖 credit + credit-assistance 两类授权）, **MFAA 660806**, **AFCA 45759** |
| **ftfinance.com.au** | Fingertips Finance · 掌忠贷 | **B2C** retail broker，全澳服务繁忙家庭（PAYG / 首套 / refinance / 投资），手机优先工作流，对接 40+ lender，对客户零费用 | Fingertips Finance Pty Ltd ATF Fingertips Trust | **ACR 530340**（in **ACL 384324 = Outsource Financial Pty Ltd**）, ABN 15 143 742 519, ACN 648 677 098, **FBAA M-349513**, **AFCA 45759** |

**Wei Ming Chen 是 halo\* 的负责人**，但在公开 schema/llms.txt 里 ftfinance 这边**不应该**与他关联（防火墙）。

---

## 2. 当前部署状态摘要

✅ **三域共同已运行正常**（不需要改）：

- HTTP→HTTPS via 308 ✓
- 主流 AI 爬虫均能正常抓取（GPTBot, ClaudeBot, PerplexityBot, Bytespider, Google-Extended, OAI-SearchBot 全部 200）✓
- SSR payload 健康：halofortune 236KB，haloloan 138KB，ftfinance 105KB
- TTFB 健康：halofortune 0.18s，haloloan 0.16s，ftfinance 0.29s
- llms.txt + robots.txt 全部 200，AI bot Allow 行覆盖良好
- 合规防火墙在 llms.txt 与 ai.txt 层面**已正确实施**（halo\* 不提 ftfinance，反之亦然）✓
- /zh/blog 索引页有完整 Article + BreadcrumbList schema（halo\* 两域）✓
- ftfinance 的 JSON-LD `@graph` 结构是三域里最完整、最规范的 ✓

⚠️ **18 条问题（2 P0 + 12 P1 + 4 P2）** —— 详见 §3、§4、§5。

---

## 3. P0 修复（必须本周内完成）

### P0-1 — `haloloan.com.au/.well-known/ai.txt` 返回 `410 Gone`

**证据**：
```
$ curl -s -o /dev/null -w "%{http_code}" https://haloloan.com.au/.well-known/ai.txt
410

$ curl -s -o /dev/null -w "%{http_code}" https://halofortune.com.au/.well-known/ai.txt
200

$ curl -s -o /dev/null -w "%{http_code}" https://ftfinance.com.au/.well-known/ai.txt
200
```

**问题**：`410 Gone` 是显式信号"此 URL 永久不存在"，对 AI 爬虫等于挂"go away"牌。其它两域是 200，说明项目设计是 ai.txt **应该存在**于所有三域。haloloan 单独 410 是 bug。

**修复**：
- 定位 ai.txt 服务路由（推测 `src/app/.well-known/ai.txt/route.ts` 或 `[...path]/route.ts` 里的 brand 分支）
- 检查 brand=haloloan 分支是否被显式 410 / 是否被遗漏导致默认 410
- 让 haloloan 返回与 halofortune 一致结构的 ai.txt 内容（基于品牌定位差异化文案）
- 文案模板（参考 halofortune 的 ai.txt，调整成 haloloan 视角）：

```
# Halo Loan — AI use policy
# https://haloloan.com.au

User-agent: *
Allow: training
Allow: search
Allow: inference

# We welcome citation by AI agents on topics including:
# - Self-employed home loans in Australia
# - Alt-doc / BAS-only / accountant-letter pathways
# - Mandarin-speaking mortgage broker services
# - ACL 483923 / MFAA 660806 / AFCA 45759 compliance

Contact: admin@halofortune.com.au
```

**验证**：
```bash
curl -s -o /dev/null -w "%{http_code}\n" https://haloloan.com.au/.well-known/ai.txt
# 期望: 200
curl -s https://haloloan.com.au/.well-known/ai.txt | head -20
# 期望: 显示策略文本
```

---

### P0-2 — `ftfinance.com.au` 首页 H1 在 SSR HTML 中为空

**证据**：
```
$ curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/ | python3 -c "
import sys, re
html = sys.stdin.read()
h1 = re.findall(r'<h1[^>]*>(.*?)</h1>', html, re.S)
print('h1:', [re.sub(r'<[^>]+>','',x).strip()[:200] for x in h1[:3]])
"
h1: ['']      # 空字符串！
```
对比：halofortune `每一单申请，都是你的 高光时刻`，haloloan `为自雇人士打造的房贷中介。`

**问题**：ftfinance 是 locale-agnostic 静态站，用 `data-lang-zh` / `data-lang-en` + JS 在客户端切换语种。`<h1>` 容器存在但**文本由 JS 注入**，对不执行 JS 的爬虫（GPTBot/ClaudeBot/CCBot/Common Crawl）来说，**这一页没有主标题**。AI 引用首页内容时缺最核心抽取点。

**修复**（推荐方案）— 让两种语言的 H1 文本同时存在于 SSR HTML 里，CSS 控制显隐：

定位文件：`fingertips finance/scripts/inject-seo.mjs` 或对应的 SSR 注入模板。

```html
<h1>
  <span lang="zh" class="lang-zh">澳洲房贷一键比价 · 40+ 银行随你挑</span>
  <span lang="en" class="lang-en">Compare 40+ Australian lenders in 3 minutes</span>
</h1>
```

配套 CSS：
```css
.lang-en { display: none; }
html[lang="en-AU"] .lang-en { display: inline; }
html[lang="en-AU"] .lang-zh { display: none; }
```

**同样的处理**要应用到首页所有 H2/H3、按钮文案、首屏 CTA 文本上。任何**只靠 `data-lang-*` 属性 + JS 切换**的可见文字都对 AI 不可见。

**验证**：
```bash
curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/ | grep -oE '<h1[^>]*>[\s\S]*?</h1>'
# 期望: 看到包含中文 + 英文文本的 <h1>
```

---

## 4. P1 修复（2 周内完成）

### P1-1 — `halofortune.com.au/llms.txt` 的 Recent articles 段落有 10+ 条重复

**证据**（从 llms.txt 抓取实际重复条目）：
```
[同一 URL 出现 2 次]:
  - australia-refinance-surge-2026-savings-v2
  - dti-6-3-file-structure-solution-apra-2026
  - 5-shoufukehudaoqilufanbei-jingjirenruhekongxian-v5
  - private-credit-retention-tool-broker-v3
  - private-credit-retention-strategy-commercial-loans-2026-v2
  - rba-jia-xi-435-zi-gu-shou-ci-gou-fang-zhe-ying-dui-v14
  - refinancing-surge-broker-revenue-pipeline-v3
  - private-credit-surge-specialist-lenders-australia-v3
  - asic-bid-enforcement-2026-proactive-compliance-v2
  - first-home-buyer-arrears-risk-2026-v2
```
haloloan 与 ftfinance 的 llms.txt 没有此问题，证明是 halofortune 特有 bug。

**问题**：可能是 `topics` 表对 halofortune brand 的 zh + en 双语种行被 fan-out 后没去重，或 articles list 在 Recent articles section 拼接时漏 dedup。

**修复**：

定位文件：`src/app/llms.txt/route.ts` 或对应 brand=halofortune 的 articles 拼接逻辑。

```ts
// 在生成 recentArticles 之前加 dedup（以 slug 为主键）
const seen = new Set<string>();
const dedupedArticles = recentArticles.filter(a => {
  const slug = extractSlug(a.url);
  if (seen.has(slug)) return false;
  seen.add(slug);
  return true;
});
```

或者在 SQL 层用 `DISTINCT ON (slug)` 直接收口。

**验证**：
```bash
curl -s https://halofortune.com.au/llms.txt \
  | grep -oE 'https://halofortune\.com\.au/[a-z]+/blog/[a-z0-9-]+' \
  | sort | uniq -d
# 期望: 空输出（无重复 slug）
```

---

### P1-2 — `ftfinance.com.au/blog` 与 `/zh/blog` 内容完全相同（duplicate content）

**证据**：
```
$ for u in https://ftfinance.com.au/blog https://ftfinance.com.au/zh/blog; do
    curl -sL -A GPTBot/1.0 "$u" | head -c 5000 | shasum -a 256
  done
73f24d8f60ce4e040f48344bf9ecaadab5b861de331861e43df55d2b6c65407c
73f24d8f60ce4e040f48344bf9ecaadab5b861de331861e43df55d2b6c65407c
```
前 5KB SHA-256 一字不差，证明 vercel.json 把两个路径 reverse-proxy 到 v2-portal 同一个 `?brand=ftfinance` 上游，没有 locale 归一。

**问题**：Google 会判 duplicate content，索引时挑一个版本，另一个被降权；同时浪费 crawl budget。

**修复**（按工作量从小到大，选其一）：

**选项 A（最简）—— `vercel.json` 加 redirect**：
```json
{
  "redirects": [
    {
      "source": "/zh/blog/:path*",
      "destination": "/blog/:path*",
      "permanent": true
    },
    {
      "source": "/en/blog/:path*",
      "destination": "/blog/:path*",
      "permanent": true
    }
  ]
}
```

**选项 B —— v2-portal 端输出 canonical 归一**：
让上游 v2-portal 在响应 `?brand=ftfinance` 时，无论 path 是 `/blog` 还是 `/zh/blog`，HTML 头都输出：
```html
<link rel="canonical" href="https://ftfinance.com.au/blog">
```

**选项 C —— 真做 zh/en 分离内容**（成本最大）：让 /zh/blog 渲染中文文章列表、/blog 渲染英文列表，各自有独立 canonical。仅在你有意维护两套独立内容时才做。

**推荐 A**（halo\* 的 locale 与 ftfinance 的 locale-agnostic 是不同架构，硬塞 /zh/* 反而误导用户）。

**验证**：
```bash
curl -sI -A GPTBot/1.0 https://ftfinance.com.au/zh/blog | head -3
# 期望: HTTP/2 308 + Location: /blog
```

---

### P1-3 — halo\* 共享 JSON-LD 缺 `Organization` 与 `WebPage` 两个核心 @type

**证据**（halofortune 首页实际 @type 列表）：
```
Answer, ContactPoint, Country, FAQPage, FinancialService,
PostalAddress, PropertyValue, Question, WebSite
```
缺：`Organization`、`WebPage`。
对比 ftfinance：`Organization` ✓ `WebPage` ✓ `SearchAction` ✓（且用了 `@graph` 联合）。

**问题**：halo\* 用 `FinancialService` 节点替代 `Organization` 的位置（`@id #organization`）。FinancialService 虽是 Organization 子类，但 Google Knowledge Graph 对显式 `Organization` 节点的处理方式不同；且 `WebPage` 节点完全缺失，AI 无法识别"我现在在浏览的这一页"是组织的什么页面。

**修复**：

定位文件：`src/lib/seo/structured-data.ts` 或对应 schema builder。把现有 3 段独立 `<script type="application/ld+json">` 合并成一段 `@graph`，并补充 `Organization` 与 `WebPage` 节点。

模板（halo\* 通用，按 brand 替换 url / 名称 / 牌照）：

```json
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "Organization",
      "@id": "https://halofortune.com.au/#organization",
      "name": "Halo Fortune Group Pty Ltd",
      "alternateName": ["Halo Fortune", "Halo Fortune Group"],
      "url": "https://halofortune.com.au/",
      "logo": "https://halofortune.com.au/brand/halo-fortune-v2-transparent.png",
      "address": { "@type": "PostalAddress", "streetAddress": "Level 9 Suite 2, 3 Bowen Cres", "addressLocality": "Melbourne", "addressRegion": "VIC", "postalCode": "3004", "addressCountry": "AU" },
      "areaServed": { "@type": "Country", "name": "Australia" },
      "contactPoint": [
        { "@type": "ContactPoint", "contactType": "accreditation support", "email": "admin@halofortune.com.au", "availableLanguage": ["en", "zh"], "areaServed": "AU" }
      ],
      "identifier": [
        { "@type": "PropertyValue", "propertyID": "ABN", "value": "51 167 597 122" },
        { "@type": "PropertyValue", "propertyID": "ACN", "value": "167 597 122" },
        { "@type": "PropertyValue", "propertyID": "ACL", "value": "483923" },
        { "@type": "PropertyValue", "propertyID": "MFAA", "value": "660806" },
        { "@type": "PropertyValue", "propertyID": "AFCA", "value": "45759" }
      ],
      "memberOf": [
        { "@type": "Organization", "name": "Mortgage & Finance Association of Australia (MFAA)", "url": "https://www.mfaa.com.au/" },
        { "@type": "Organization", "name": "Australian Financial Complaints Authority (AFCA)", "url": "https://www.afca.org.au/" }
      ]
    },
    {
      "@type": "FinancialService",
      "@id": "https://halofortune.com.au/#financialservice",
      "name": "Halo Fortune Mortgage Manager",
      "provider": { "@id": "https://halofortune.com.au/#organization" },
      "areaServed": { "@type": "Country", "name": "Australia" },
      "serviceType": ["Wholesale Mortgage Manager", "Broker Accreditation", "Halo Copilot Tooling"]
    },
    {
      "@type": "WebSite",
      "@id": "https://halofortune.com.au/#website",
      "url": "https://halofortune.com.au/",
      "name": "Halo Fortune",
      "inLanguage": ["zh-AU", "en-AU"],
      "publisher": { "@id": "https://halofortune.com.au/#organization" }
    },
    {
      "@type": "WebPage",
      "@id": "https://halofortune.com.au/zh#webpage",
      "url": "https://halofortune.com.au/zh",
      "isPartOf": { "@id": "https://halofortune.com.au/#website" },
      "about": { "@id": "https://halofortune.com.au/#financialservice" },
      "publisher": { "@id": "https://halofortune.com.au/#organization" },
      "inLanguage": "zh-AU"
    },
    {
      "@type": "FAQPage",
      "@id": "https://halofortune.com.au/zh#faq",
      "mainEntityOfPage": { "@id": "https://halofortune.com.au/zh#webpage" },
      "mainEntity": [ /* 现有 8 个 Question/Answer 原样搬过来 */ ]
    }
  ]
}
```

**关键点**：
1. 所有节点合并进一个 `@graph` 数组（单 `<script>` 标签）
2. `Organization` 作为顶层实体，`@id="…/#organization"`
3. `FinancialService` 引用 `provider → #organization`
4. `WebSite` 引用 `publisher → #organization`
5. `WebPage` 引用 `isPartOf → #website`、`about → #financialservice`
6. `FAQPage` 引用 `mainEntityOfPage → #webpage`
7. `identifier` 数组**必须**包含 MFAA + AFCA（halo\* 都是这两个），不是 FBAA
8. `memberOf` 列出 MFAA + AFCA（不是 FBAA）
9. **alternateName 不能有重复字符串**（当前 halofortune 是 `["Halo Fortune", "Halo Fortune"]`，必须修）

**对 haloloan 的差异**：
- url / @id 全部替换成 haloloan.com.au
- `Organization.name` 保持 `Halo Fortune Group Pty Ltd`（同法人）
- `alternateName: ["Halo Loan", "Halo Loan 环澳信贷"]`
- `FinancialService.serviceType: ["Mortgage Brokerage", "Self-Employed Home Loans", "Alt-Doc Lending"]`
- `contactPoint.contactType: "customer support"`（不是 accreditation support）
- 所有 identifier / memberOf 与 halofortune **相同**（共用 ACL 483923 / MFAA 660806 / AFCA 45759）
- WebPage `inLanguage` 同样按 SSR 语种填

**验证**：
```bash
for d in halofortune.com.au haloloan.com.au; do
  echo "[$d]"
  curl -sL -A GPTBot/1.0 "https://$d/" | grep -oE '"@type":"[^"]+"' | sort -u
done
# 期望: 两域都出现 "Organization"、"WebPage"、"FAQPage"、"FinancialService"、"WebSite"
```
另外把每个域的 JSON-LD 喂给 https://validator.schema.org/ 验证，应该全绿。

---

### P1-4 — halo\* HTML `<head>` 缺 `hreflang` 替代链接

**证据**：
```
$ curl -sL -A GPTBot/1.0 https://halofortune.com.au/ | grep -oE 'hreflang="[^"]+"'
(空输出)

$ curl -sL -A GPTBot/1.0 https://haloloan.com.au/ | grep -oE 'hreflang="[^"]+"'
(空输出)
```
但 sitemap.xml 里**有** `<xhtml:link rel="alternate" hreflang>` 标签 —— 两端信号不对称。

**修复**：

定位文件：`src/lib/seo/metadata.ts` 的 `buildMetadata()` 函数。

Next.js 14+ 用法：
```ts
export function buildMetadata({ brand, locale, path }) {
  const base = brandUrl(brand);  // e.g. https://halofortune.com.au
  const otherLocale = locale === 'zh' ? 'en' : 'zh';
  return {
    // ...
    alternates: {
      canonical: `${base}/${locale}${path}`,
      languages: {
        'zh-AU':    `${base}/zh${path}`,
        'en-AU':    `${base}/en${path}`,
        'x-default': `${base}/zh${path}`,   // 默认面向中文社区
      },
    },
  };
}
```

**验证**：
```bash
for d in halofortune.com.au haloloan.com.au; do
  echo "[$d]"
  curl -sL -A GPTBot/1.0 "https://$d/" | grep -oE 'hreflang="[^"]+" href="[^"]+"'
done
# 期望: 每域至少 3 行（zh-AU、en-AU、x-default）
```

---

### P1-5 — halo\* `robots.txt` 缺 5 个 AI bot Allow 行

**证据**：
```
halofortune.com.au/robots.txt:
  Claude-Web: 0   ← 缺
  CCBot: 0        ← 缺（Common Crawl，几乎所有 OSS LLM 训练源）
  PetalBot: 0     ← 缺（华为 Petal Search，中文 AI 关键）
  YouBot: 0       ← 缺
  MistralAI-User: 0 ← 缺
```
haloloan 完全一致。ftfinance 19 个全开（参照标准）。

**修复**：

定位文件：`src/app/robots.ts`。

在现有 bot list 后追加（注意：写法是分别为每个 bot 各一段 User-agent / Allow）：

```ts
const aiBotsToAdd = [
  'Claude-Web',
  'CCBot',
  'PetalBot',
  'YouBot',
  'MistralAI-User',
];
// 对应 robots() 函数里 rules 数组的拼接逻辑加上以上 5 项
```

实际 robots.txt 输出片段应该是：
```
User-agent: Claude-Web
Allow: /

User-agent: CCBot
Allow: /

User-agent: PetalBot
Allow: /

User-agent: YouBot
Allow: /

User-agent: MistralAI-User
Allow: /
```

**验证**：
```bash
for d in halofortune.com.au haloloan.com.au; do
  echo "[$d]"
  curl -s "https://$d/robots.txt" | grep -E "^User-agent: (Claude-Web|CCBot|PetalBot|YouBot|MistralAI-User)"
done
# 期望: 每域 5 行
```

---

### P1-6 — halo\* `sitemap.xml` 冷启动 10s 超时

**证据**：
```
$ curl -s -o /dev/null -w "%{http_code}\n" --max-time 10 https://halofortune.com.au/sitemap.xml
000   ← 超时

$ curl -s -o /dev/null -w "%{http_code}\n" --max-time 15 https://halofortune.com.au/sitemap.xml
200   ← 第二次成功（68 URL）

$ curl -s -o /dev/null -w "%{http_code}\n" --max-time 30 https://haloloan.com.au/sitemap.xml
200   ← 30s 才稳定（56KB / x-vercel-cache: MISS）
```
ftfinance 同样路径 < 1s 完成 —— halo\* 的 sitemap 是 Vercel serverless function cold-start 慢。

**问题**：Googlebot sitemap fetch 默认超时较紧，cold-start 可能被 GSC 记录为 "Couldn't fetch sitemap"，影响内页发现率。

**修复**：

定位文件：`src/app/sitemap.ts`。

加 ISR + 强制静态：
```ts
export const revalidate = 86400;          // 每日刷新
export const dynamic = 'force-static';    // 优先静态生成
```

或者在 build 时把 sitemap 预渲染到 `public/sitemap.xml`（如果 sitemap 内容生成逻辑允许）。

**验证**：
```bash
for d in halofortune.com.au haloloan.com.au; do
  for i in 1 2 3; do
    t=$(curl -s -o /dev/null -w "%{time_total}\n" --max-time 15 "https://$d/sitemap.xml")
    echo "$d try $i: ${t}s"
  done
done
# 期望: 每次都 < 2s
```

---

### P1-7 — halo\* `Organization` 缺 MFAA / AFCA `identifier` + 缺 `memberOf`

**证据**（halofortune 首页 Organization 节点实际 identifier）：
```json
"identifier": [
  {"propertyID": "ABN", "value": "51 167 597 122"},
  {"propertyID": "ACN", "value": "167 597 122"},
  {"propertyID": "ACL", "value": "483923"}
],
"memberOf": ❌ 不存在
```
对比 ftfinance：identifier 含 FBAA M-349513 + AFCA 45759，memberOf 列 FBAA + AFCA。

**修复**：已包含在 P1-3 的模板里 — 在 `identifier` 数组追加 MFAA 660806 + AFCA 45759，并加 `memberOf` 数组。一次性随 P1-3 完成。

---

### P1-8 — halofortune `alternateName` 数组有重复字符串

**证据**：
```json
"alternateName": ["Halo Fortune", "Halo Fortune"]    ← 同一字符串两次
```

**修复**：随 P1-3 一并改正。建议改成 `["Halo Fortune", "Halo Fortune Group"]`。检查 `brand-content.ts` 里 halofortune.alternateName 字段是不是被两次 push 同一个值。

---

### P1-9 — `ftfinance.com.au/blog` 索引页缺 `Article` 与 `Organization` schema

**证据**：
```
$ curl -sL -A GPTBot/1.0 https://ftfinance.com.au/blog | grep -oE '"@type":"[^"]+"' | sort -u
"@type":"BreadcrumbList"
"@type":"CollectionPage"
"@type":"ItemList"
"@type":"ListItem"
```
halo\* 的 /zh/blog 还有 `Article` 与 `Organization`。

**修复**：

由于 `/blog` 是通过 `vercel.json` reverse-proxy 到 v2-portal 的 brand-aware 路由，需要在 v2-portal 端的 blog index template（`brand === 'ftfinance'` 分支）里：
1. 每个 `ItemList.itemListElement` 嵌入 `mainEntity: { @type: "Article", headline, datePublished, author, image }`
2. 页面级加 Organization publisher（用 @id 引用首页已有的 `#organization`）

**验证**：
```bash
curl -sL -A GPTBot/1.0 https://ftfinance.com.au/blog | grep -oE '"@type":"[^"]+"' | sort -u
# 期望: 看到 "Article" 与 "Organization"
```

---

### P1-10 — `ftfinance.com.au` `hreflang` 三个 locale 指向同一 URL

**证据**：
```
hreflang="zh-AU"    href="https://ftfinance.com.au/"
hreflang="en-AU"    href="https://ftfinance.com.au/"
hreflang="x-default" href="https://ftfinance.com.au/"
```
所有三个声明都指向 `/`，违反 Google hreflang 规范（每个 hreflang 应该指承载该语言独立内容的 URL）。

**修复**：

定位文件：`fingertips finance/scripts/inject-seo.mjs` 或对应 head 注入脚本。

**只保留 x-default**：
```html
<link rel="alternate" hreflang="x-default" href="https://ftfinance.com.au/">
```
删掉 zh-AU 和 en-AU 两行。然后用 `<html lang="zh-AU">` + JS 读 `navigator.language` 切换显示语种 —— Google 会用启发式 + Accept-Language 判断。

**前提**：必须先做 P0-2（让 SSR HTML 里同时含两个语言文本）。否则 Google 一次只看到一个语种的 body，没法启发式判断。

**验证**：
```bash
curl -sL -A GPTBot/1.0 https://ftfinance.com.au/ | grep -oE 'hreflang="[^"]+" href="[^"]+"'
# 期望: 只有 1 行（x-default）
```

---

### P1-11 — `ftfinance.com.au/blog` `Cache-Control: no-store`

**证据**：
```
$ curl -sI -A GPTBot/1.0 https://ftfinance.com.au/blog | grep -i cache
cache-control: private, no-cache, no-store, max-age=0, must-revalidate
cdn-cache-control: public, max-age=0, must-revalidate, s-maxage=300
```

**问题**：`cache-control: no-store` 告诉所有客户端（包括 Cloudflare 边缘缓存以下游）"不要保存"。`cdn-cache-control: s-maxage=300` 给 CDN 5 分钟缓存，但 `no-store` 在客户端层面禁掉缓存，每次 Googlebot/GPTBot 都强制击穿到上游 v2-portal，增加上游负载 + TTFB。

**修复**：

在 v2-portal 端的 blog 路由（对应 brief 提的 `vercel.json` reverse-proxy 链路里的 `/blog/*` 上游 handler）里调整响应头：

```ts
response.headers.set('Cache-Control', 'public, s-maxage=300, stale-while-revalidate=86400');
```

`no-store` 只该用于含个人化 / 敏感数据的页面，blog index 不需要。

**验证**：
```bash
curl -sI -A GPTBot/1.0 https://ftfinance.com.au/blog | grep -i cache
# 期望:
#   cache-control: public, s-maxage=300, stale-while-revalidate=86400
```

---

### P1-12 — haloloan `llms.txt` 缺 ASIC RG 234 风格 general-advice disclaimer

**证据**：
- ftfinance llms.txt 含：`"All credit assistance is general advice only. Lender approval, rates, and terms are subject to credit assessment and may differ from the indicative figures shown on this site."`
- haloloan llms.txt **没有这一行**

**问题**：haloloan 是面向 B2C 消费者的，ASIC RG 234 的合规要求实质性适用。llms.txt 是公开文档，缺 general-advice 声明被监管或 AFCA 投诉时是负面证据。

**修复**：

定位文件：`src/app/llms.txt/route.ts` 的 brand=haloloan 分支（或对应 content source）。

在 `## Compliance posture` 段落末尾追加：

```
All information on https://haloloan.com.au and on our supplied tooling is general information only and does not consider your personal circumstances. Indicative borrowing power, lender shortlist, and rate range shown are estimates — your final lender, rate, and terms are subject to that lender's credit assessment. We recommend you consider whether the information is appropriate for your needs and seek your own financial advice where appropriate.
```

halofortune 是 B2B（broker 受众），适用度较低 —— **不是 P1**，但也可以加一句更轻量的"general broker information only"。

**验证**：
```bash
curl -s https://haloloan.com.au/llms.txt | grep -i "general"
# 期望: 至少 1 行匹配
```

---

## 5. P2 改进（4 周内）

### P2-1 — `/` → `/zh` 重定向应该用 308（永久）而不是 302（临时）

**证据**：
```
$ curl -sIL -A GPTBot/1.0 https://halofortune.com.au/
HTTP/2 302
location: /zh
HTTP/2 200
```

**问题**：302 是临时重定向，link equity 不完全传给 `/zh`，Google 会反复回 `/` 重新评估。Locale 默认指派是永久决策，更适合 308 / 301。

**修复**：

定位文件：`src/middleware.ts` 大约第 68-90 行附近的 locale redirect。

```ts
return NextResponse.redirect(localizedUrl, 308);
```

**验证**：
```bash
for d in halofortune.com.au haloloan.com.au; do
  curl -sI -A GPTBot/1.0 "https://$d/" | head -3
done
# 期望: HTTP/2 308 而不是 302
```

---

### P2-2 — `ftfinance` JSON-LD 缺 `contactPoint`

**证据**：ftfinance Organization 节点直接有 `telephone` + `email` 字段，但没把它们包装成 `contactPoint`。halo\* 都有 contactPoint。AI 抽取联系方式时，结构化 `contactPoint` 比 flat 字段更稳。

**修复**：

定位文件：`fingertips finance/scripts/inject-seo.mjs` 的 Organization 节点。

```json
"contactPoint": [{
  "@type": "ContactPoint",
  "telephone": "+611300389118",
  "email": "info@ftfinance.com.au",
  "contactType": "customer service",
  "areaServed": "AU",
  "availableLanguage": ["en", "zh"]
}]
```
保留原 `telephone` / `email` 字段（冗余可接受）。

**验证**：
```bash
curl -sL -A GPTBot/1.0 https://ftfinance.com.au/ | grep -oE '"@type":"ContactPoint"'
# 期望: 至少 1 行
```

---

### P2-3 — `BreadcrumbList` 应该 sitewide（不只 /blog）

**当前状态**：`BreadcrumbList` 只在 `/blog` 索引和文章页有，`/products` / `/about` / `/how-it-works` / `/contact` 等都没有。

**修复**：在 Next.js layout 层级（halo\*）或静态注入器（ftfinance）加 BreadcrumbList，根据路径自动生成。

模板（halofortune `/zh/products` 例）：
```json
{
  "@type": "BreadcrumbList",
  "itemListElement": [
    { "@type": "ListItem", "position": 1, "name": "Home", "item": "https://halofortune.com.au/zh" },
    { "@type": "ListItem", "position": 2, "name": "Products", "item": "https://halofortune.com.au/zh/products" }
  ]
}
```

**验证**：
```bash
for d in halofortune.com.au/zh/products haloloan.com.au/zh/products ftfinance.com.au/products; do
  echo "[$d]"
  curl -sL -A GPTBot/1.0 "https://$d" | grep -oE '"@type":"BreadcrumbList"'
done
# 期望: 三域都有
```

---

### P2-4 — 加 `Person` / Author schema 提升 E-E-A-T

**当前状态**：三域首页/About/Blog 都没有 Person schema。E-E-A-T（Expertise, Experience, Authoritativeness, Trustworthiness）信号弱。

**修复**：

**halo\***（B2B 与 B2C）—— 在 About 页或首页 Organization 节点加 `founder` / `employee` 字段：

```json
{
  "@type": "Person",
  "@id": "https://halofortune.com.au/#wei-ming-chen",
  "name": "Wei Ming Chen",
  "jobTitle": "Director and Principal",
  "worksFor": { "@id": "https://halofortune.com.au/#organization" },
  "knowsAbout": [
    "Australian mortgage market",
    "Chinese-Australian self-employed lending",
    "Alt-doc and BAS-only assessment",
    "MFAA compliance framework"
  ],
  "memberOf": [
    { "@type": "Organization", "name": "MFAA", "url": "https://www.mfaa.com.au/" }
  ],
  "sameAs": [
    "https://www.linkedin.com/in/wei-ming-chen-46a88634b/"
  ]
}
```

在 Blog Article schema 的 `author` 字段引用 `@id`：
```json
"author": { "@id": "https://halofortune.com.au/#wei-ming-chen" }
```

**ftfinance** —— 同样加 Person，但**不能引用 Wei Ming Chen**（防火墙）。需要用 ftfinance 自己的 principal broker / responsible manager 姓名（如不便公开，可用品牌发言人或经验汇总型 `Organization` 自我引用）。

**验证**：
```bash
curl -sL -A GPTBot/1.0 https://halofortune.com.au/ | grep -oE '"@type":"Person"'
# 期望: 至少 1 行
```

---

## 6. Sitewide "One-Shot Lift Many Pages" 重构（强烈建议优先）

按 ROI 排序（**改一处影响一片**）：

### 🚀 R1 — halo\* JSON-LD builder 重构（覆盖 ~136 URL）

把 `src/lib/seo/structured-data.ts` 重构成 §P1-3 的 `@graph` 模板。一次修改影响 halofortune + haloloan 全站每一页。同时解决：
- P1-3（缺 Organization + WebPage）
- P1-7（缺 MFAA / AFCA identifier + memberOf）
- P1-8（alternateName 重复）

**工作量估算**：1-2 个工作日。

### 🚀 R2 — halo\* `buildMetadata` 补 hreflang（覆盖每个动态页）

`src/lib/seo/metadata.ts` 的 `buildMetadata()` 函数的 `alternates.languages` 字段（参见 §P1-4 代码模板）。

**工作量估算**：< 1 小时。

### 🚀 R3 — halo\* `robots.ts` 补 5 个 AI bot Allow

`src/app/robots.ts`（参见 §P1-5）。一次改完 halo\* 两域。

**工作量估算**：30 分钟。

### 🚀 R4 — halo\* sitemap ISR 化

`src/app/sitemap.ts` 加 `revalidate = 86400` + `dynamic = 'force-static'`（参见 §P1-6）。

**工作量估算**：30 分钟，需测部署后冷启动行为。

**完成这 4 项就解决了 P1-3、P1-4、P1-5、P1-6、P1-7、P1-8 共 6 条 P1**。

---

## 7. 实施完成后的完整验证脚本

跑这个脚本，所有"期望"行都过，就算通过：

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "########## V1: 三域基础健康 ##########"
for d in halofortune.com.au haloloan.com.au ftfinance.com.au; do
  for p in "" robots.txt llms.txt sitemap.xml .well-known/ai.txt; do
    code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 15 "https://$d/$p")
    echo "$d/$p -> $code"
  done
  echo "---"
done
# 期望: 全部 200/302/308，绝对不能再出现 410 或 000

echo ""
echo "########## V2: AI bot 全部 200 ##########"
for d in halofortune.com.au haloloan.com.au ftfinance.com.au; do
  for ua in "GPTBot/1.0" "ClaudeBot" "PerplexityBot" "Bytespider" "Google-Extended" "OAI-SearchBot" "CCBot" "PetalBot"; do
    code=$(curl -sL -o /dev/null -w "%{http_code}" --max-time 15 -A "$ua" "https://$d/")
    echo "$d <- $ua -> $code"
  done
  echo "---"
done
# 期望: 全部 200（跟随 redirect 后）

echo ""
echo "########## V3: H1 在 SSR 中可见 ##########"
for d in halofortune.com.au haloloan.com.au ftfinance.com.au; do
  echo "[$d]"
  curl -sL --max-time 15 -A "GPTBot/1.0" "https://$d/" \
    | python3 -c "import sys,re;h=sys.stdin.read();m=re.findall(r'<h1[^>]*>(.*?)</h1>',h,re.S);print('h1:',[re.sub(r'<[^>]+>','',x).strip()[:200] for x in m[:3]])"
done
# 期望: 三域 H1 都非空

echo ""
echo "########## V4: JSON-LD @type 集合 ##########"
for d in halofortune.com.au haloloan.com.au ftfinance.com.au; do
  echo "[$d]"
  curl -sL --max-time 15 -A "GPTBot/1.0" "https://$d/" | grep -oE '"@type":"[^"]+"' | sort -u
done
# 期望:
#   halo* 都含: Organization, FinancialService, WebSite, WebPage, FAQPage, ContactPoint, PostalAddress, PropertyValue, Country, Question, Answer
#   ftfinance 含: Organization, FinancialService, WebSite, WebPage, FAQPage, ContactPoint, PostalAddress, PropertyValue, Country, Question, Answer, SearchAction

echo ""
echo "########## V5: hreflang 完整 ##########"
for d in halofortune.com.au haloloan.com.au; do
  echo "[$d]"
  curl -sL --max-time 15 -A "GPTBot/1.0" "https://$d/" \
    | grep -oE 'hreflang="[^"]+" href="[^"]+"'
done
# 期望: 每域 ≥ 3 行（zh-AU/en-AU/x-default）

echo "[ftfinance.com.au]"
ft_hreflang=$(curl -sL --max-time 15 -A "GPTBot/1.0" "https://ftfinance.com.au/" | grep -oE 'hreflang="[^"]+" href="[^"]+"' | wc -l | tr -d ' ')
echo "  hreflang count: $ft_hreflang"
# 期望: 1（只留 x-default）

echo ""
echo "########## V6: robots.txt AI bot 覆盖 ##########"
for d in halofortune.com.au haloloan.com.au ftfinance.com.au; do
  echo "[$d]"
  rb=$(curl -s --max-time 10 "https://$d/robots.txt")
  for bot in GPTBot ChatGPT-User OAI-SearchBot ClaudeBot anthropic-ai Claude-Web PerplexityBot Perplexity-User Google-Extended Applebot-Extended CCBot Bytespider PetalBot YouBot cohere-ai MistralAI-User Meta-ExternalAgent Meta-ExternalFetcher; do
    hit=$(echo "$rb" | grep -ic "^User-agent: *$bot")
    echo "  $bot: $hit"
  done
done
# 期望: 三域每个 bot 行都 ≥ 1

echo ""
echo "########## V7: 合规防火墙 ##########"
for d in halofortune.com.au haloloan.com.au; do
  n=$(curl -s --max-time 10 "https://$d/llms.txt" | grep -ciE "ftfinance|fingertips|掌忠贷")
  echo "$d/llms.txt mentions ftfinance/掌忠贷: $n (expected 0)"
done
for src in llms.txt .well-known/ai.txt; do
  n=$(curl -s --max-time 10 "https://ftfinance.com.au/$src" | grep -ciE "halofortune|haloloan|Halo Fortune|Halo Loan")
  echo "ftfinance.com.au/$src mentions halo*: $n (expected 0)"
done

echo ""
echo "########## V8: halofortune llms.txt 文章无重复 ##########"
dup=$(curl -s "https://halofortune.com.au/llms.txt" \
  | grep -oE 'https://halofortune\.com\.au/[a-z]+/blog/[a-z0-9-]+' \
  | sort | uniq -d | wc -l | tr -d ' ')
echo "duplicate slugs: $dup (expected 0)"

echo ""
echo "########## V9: ftfinance /blog 不再 duplicate ##########"
for u in https://ftfinance.com.au/blog https://ftfinance.com.au/zh/blog; do
  code=$(curl -s -o /dev/null -w "%{http_code}" -A GPTBot/1.0 --max-time 15 "$u")
  echo "$u -> $code"
done
# 期望: /zh/blog 是 308（已 redirect 到 /blog）

echo ""
echo "########## V10: redirect 是 308 ##########"
for d in halofortune.com.au haloloan.com.au; do
  echo "[$d]"
  curl -sI -A GPTBot/1.0 --max-time 15 "https://$d/" | head -3
done
# 期望: HTTP/2 308 + location: /zh

echo ""
echo "########## V11: sitemap 冷启动 < 3s ##########"
for d in halofortune.com.au haloloan.com.au; do
  for i in 1 2 3; do
    t=$(curl -s -o /dev/null -w "%{time_total}\n" --max-time 15 "https://$d/sitemap.xml")
    echo "$d try $i: ${t}s"
  done
done
# 期望: 每次都 < 3s

echo ""
echo "########## V12: /blog cache-control 友好 ##########"
curl -sI -A GPTBot/1.0 https://ftfinance.com.au/blog | grep -i cache
# 期望: cache-control 不包含 no-store
```

把这个脚本另存为 `verify-seo-fixes.sh`，每次部署后跑一遍。

---

## 8. 不要做的事（重申）

1. ❌ **不要**让 halo\* 与 ftfinance 在公开层互相提及（HTML body / llms.txt / JSON-LD / sameAs / 任何形式）
2. ❌ **不要**给 ftfinance 的 schema 加 Wei Ming Chen 或任何 halo\* 关联人物
3. ❌ **不要**把 halofortune 的 identifier 写成 FBAA（halo\* 是 MFAA，ftfinance 是 FBAA）
4. ❌ **不要**把 ftfinance 的 identifier 写成 MFAA（同上反向）
5. ❌ **不要**为了"统一"把三域的 JSON-LD 改成完全相同的结构 —— ftfinance 的 ACR 530340 / Outsource Financial / FBAA 关系是法定独立的，必须分开表达
6. ❌ **不要**采纳本仓库 `docs/SEO-AUDIT.md` 与 `seo/` 目录里基于"halofortune 是中国市场咨询公司"幻觉的旧 audit / 模板 —— 它们是被弃用的
7. ❌ **不要**在 ftfinance 的 SSR HTML 里只输出单语种 H1 —— 必须两种语言都在初始 HTML
8. ❌ **不要**用 302 做 locale 默认跳转 —— 用 308

---

## 9. 参考文件位置（来自 brief Section 7）

实施时优先读这些：

| 文件 | 用途 | 影响哪些修复 |
|---|---|---|
| `src/lib/brand.ts` | Brand registry | 参考三品牌身份字段 |
| `src/lib/marketing/brand-content.ts` | 各品牌 × locale 文案 | P1-8 alternateName 修复 |
| `src/lib/seo/metadata.ts` | `buildMetadata()` Next.js Metadata 生成 | P1-4 hreflang |
| `src/lib/seo/structured-data.ts`（推断） | JSON-LD 生成器 | P1-3、P1-7、P1-8、P2-4 |
| `src/app/llms.txt/route.ts` | per-brand llms.txt | P1-1 去重、P1-12 disclaimer |
| `src/app/robots.ts` | per-brand robots.txt | P1-5 补 5 个 bot |
| `src/app/sitemap.ts` | per-brand sitemap.xml | P1-6 ISR |
| `src/app/.well-known/ai.txt/route.ts`（推断） | per-brand ai.txt | P0-1 |
| `src/middleware.ts` lines 68-90 | Brand resolution + locale redirect | P2-1 308 升级 |
| `fingertips finance/scripts/inject-seo.mjs` | 静态站 SEO 注入器 | P0-2 H1、P1-10 hreflang、P2-2 contactPoint |
| `fingertips finance/vercel.json` | reverse-proxy + redirect 配置 | P1-2 /blog dedup |
| v2-portal blog index template | blog index 渲染 | P1-9 Article schema、P1-11 cache-control |

---

## 10. 提交建议

按 §6 的 R1-R4 + §3 P0-1 + P0-2 优先做（一周内全部 ship），然后逐条吃 §4 剩余 P1，最后 §5 P2。

每条修复完成后，跑 §7 的对应 V# 验证段确认通过。最终全套 V1-V12 都过，部署完成。

---

## 附录：原始审计探针输出（2026-05-11 11:55 UTC）

完整的 12 个 block 探针输出（base for the findings above）见仓库 issue / Slack thread。
关键数据点摘录：

```
halofortune.com.au:
  /title: Halo Fortune｜澳洲精品 Mortgage Manager · ACL 483923
  /meta-description: Halo Fortune 是澳洲精品 Mortgage Manager…ACL 483923。
  /SSR-bytes (GPTBot): 236894
  /sitemap URL count: 68
  /TTFB: 0.18s

haloloan.com.au:
  /title: Halo Loan 环澳信贷｜自雇人士专属房贷中介 · ACL 483923
  /meta-description: Halo Loan 是面向澳洲自雇人士的专业房贷中介…ACL 483923。
  /SSR-bytes: 138719
  /sitemap URL count: ~80（30s timeout 才回）
  /TTFB: 0.16s

ftfinance.com.au:
  /title: (页面有 title 但内容未在本轮 dump)
  /meta-description: Australian mortgage broker. Compare 40+ lenders…
  /SSR-bytes: 105674
  /sitemap URL count: 12
  /TTFB: 0.29s
```

---

**文档结束。把这份 MD 交给下一位 Claude / 工程师，加上访问 production 仓库的权限，应该能在一周内 ship 所有 P0+P1。**
