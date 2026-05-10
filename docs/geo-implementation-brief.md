# GEO 实施交接简报 · halofortune.com.au

> **致接手 Agent**：本文档是一份独立的执行简报。你不需要任何先前对话上下文。读完即可开工。

---

## 0. 你的任务（Mission）

把 https://halofortune.com.au 从"对生成式搜索引擎隐身"的状态，改造成 ChatGPT / Perplexity / Google AI Overviews / 文心一言 / Kimi 等 AI 引擎在被问及"Melbourne mortgage broker / 墨尔本华人房贷经纪"时**会主动引用的权威源**。

业务焦点（不要发散）：**贷款 / 房贷经纪（Mortgage Broker）**。其他业务线（教育中介、中国市场拓展、电商、HR）一律降级为页脚链接或迁出子域。

---

## 1. 你需要先发现的事实（Discovery, 30 分钟）

按顺序执行下列调查，把结果写到 `docs/geo-discovery-notes.md` 后再开始动手改：

```bash
# 1.1 当前主域响应
for UA in "" "Mozilla/5.0" "GPTBot/1.0" "ClaudeBot/1.0" "PerplexityBot/1.0" \
          "Google-Extended" "OAI-SearchBot/1.0" "Bytespider"; do
  printf "%-22s → " "${UA:-empty}"
  curl -sIL -A "$UA" --max-time 10 https://halofortune.com.au/ | head -1
done

# 1.2 robots.txt / sitemap 现状
curl -s https://halofortune.com.au/robots.txt
curl -s https://halofortune.com.au/sitemap.xml | head -50

# 1.3 渲染模式：HTML 是否已含正文？
curl -s -A "Mozilla/5.0 (compatible; ClaudeBot/1.0; +claudebot@anthropic.com)" \
     https://halofortune.com.au/ | grep -ciE "mortgage|home loan|broker|首付|房贷"

# 1.4 已有结构化数据？
curl -s https://halofortune.com.au/ | grep -oE 'application/ld\+json[^<]{0,500}'

# 1.5 技术栈线索
curl -sI https://halofortune.com.au/ | grep -iE "server|x-powered-by|cf-ray"
```

**清单产出**：
- 当前哪些 UA 被 403、哪些 200
- 是否有 robots.txt / sitemap.xml
- 是否 SSR（HTML 是否含关键词）
- 是 Cloudflare / Nginx / WordPress / Next.js / 其他？
- CMS 后台地址 / 部署方式（你有权限的范围）

> ⚠ 如果你**没有**生产站源码或后台访问权限，停在这里，把 Discovery 报告交给用户，不要继续往下做。

> 💡 **没有 shell 怎么办**：如果你在 CMS 后台环境无法跑 curl，用以下任一替代：(a) https://www.webpagetest.org/ 自定义 UA 测试；(b) https://search.google.com/test/mobile-friendly 看渲染后 HTML；(c) 浏览器开发者工具 → Network → 设置 User-Agent override 重放请求；(d) 用 `wget --user-agent=...` 替代。把方法和结果写入 Discovery 笔记。

---

## 2. 关键事实（已确认，可直接写入页面）

| 项目 | 值 |
|---|---|
| 公司 | Halo Fortune Group Pty Ltd（含 Sino Talent Resources / Adcess / Happy Mall / Halo HR 子品牌） |
| 地址 | 332 Kings Way, South Melbourne, VIC 3205, Australia |
| 电话 | +61 3 9999 9768 |
| 邮箱 | kevin@halofortune.com |
| 房贷经纪人 | Wei Chen |
| 业务范围 | First Home Buyer · Property Investor · Owner-Occupied · Bridging Loan |
| 目标客群 | 澳洲本地居民 + 墨尔本华人买家（中英双语） |

> 以下信息**用户必须提供**才能填空：ABN、ACL（Australian Credit Licence）或 ACR（Credit Representative）编号、MFAA/FBAA 会员编号、**AFCA 会员号**（持牌信贷机构强制要求的外部争议解决机制）、合作贷方（lender panel）名单、Wei Chen 的从业年限/累计放款规模、营业时间、地理坐标。**未提供的字段不要编造，留 TODO 占位。**

> ⚠ **澳洲合规硬性要求**（不是 GEO 但不能漏）：
> - 任何讨论利率/借款额/还款的页面，底部必须有 **ASIC RG 234 "general advice"** 免责声明，例如："The information on this page is general in nature only and does not take into account your personal objectives, financial situation or needs. Before acting on any information, consider whether it is appropriate for you and seek personal financial advice."
> - 全站底部必须显示 **ABN + ACL/ACR 编号 + AFCA 编号**，格式参考 ASIC MoneySmart 官方指引
> - 中文版需提供同等中文翻译版本

---

## 3. 执行计划（按依赖顺序）

### Phase 1 · 解除 AI 爬虫拦截（必做，1 周内，阻塞所有后续工作）

**为什么先做这个**：内容写得再好，AI 抓不到等于没写。

#### 1.1 `robots.txt`（站点根目录，新建或覆盖）

```txt
# /robots.txt
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /wp-admin/
Disallow: /api/

# Generative AI crawlers — explicit allow
User-agent: GPTBot
Allow: /

User-agent: OAI-SearchBot
Allow: /

User-agent: ChatGPT-User
Allow: /

User-agent: ClaudeBot
Allow: /

User-agent: anthropic-ai
Allow: /

User-agent: Claude-Web
Allow: /

User-agent: PerplexityBot
Allow: /

User-agent: Perplexity-User
Allow: /

User-agent: Google-Extended
Allow: /

User-agent: Applebot-Extended
Allow: /

User-agent: CCBot
Allow: /

User-agent: Bytespider
Allow: /

User-agent: Amazonbot
Allow: /

User-agent: YouBot
Allow: /

User-agent: cohere-ai
Allow: /

User-agent: Diffbot
Allow: /

Sitemap: https://halofortune.com.au/sitemap.xml
```

> 注：每个独立 `User-agent` 块在 robots.txt 协议中相互独立，不继承 `User-agent: *` 块。这里的"显式 Allow"是**意图信号**——部分 AI 爬虫（如 OpenAI 公布的政策）会专门检查命名块的存在与否再决定是否抓取。即便 `User-agent: *` 已经允许，命名块仍要写。

#### 1.1.b 新增 `/llms.txt`（站点根目录，AI 时代事实标准）

```txt
# /llms.txt
# Specification: https://llmstxt.org/

# Halo Fortune

> Melbourne-based mortgage broker (ABN [TODO] · ACL/ACR #[TODO]) helping Australian residents and Chinese-speaking buyers with first-home, investment, owner-occupied and bridging loans. Bilingual service in English and 中文.

## Core Services

- [Mortgage Broker Melbourne (Pillar)](https://halofortune.com.au/mortgage-broker-melbourne/): Topic hub covering all loan products and the broker process.
- [First Home Buyer Loans](https://halofortune.com.au/first-home-buyer/): FHOG eligibility, deposit requirements, LMI guidance.
- [Investment Property Loans](https://halofortune.com.au/investment-property-loan/): LVR, serviceability, rental income assessment.
- [Owner-Occupied Loans](https://halofortune.com.au/owner-occupied-loan/): Comparison of fixed vs variable, offset vs redraw.
- [Refinance](https://halofortune.com.au/refinance/): When to refinance, costs, lender switching.
- [Bridging Loans](https://halofortune.com.au/bridging-loan/): Buy-before-sell scenarios, peak debt management.

## About

- [Wei Chen — Mortgage Broker](https://halofortune.com.au/about/wei-chen/): Credentials, MFAA/FBAA membership, languages.
- [Contact](https://halofortune.com.au/contact/): 332 Kings Way, South Melbourne · +61 3 9999 9768.

## 中文

- [中文首页](https://halofortune.com.au/zh/)
```

新增同时输出 `/llms-full.txt`（每个核心页的 markdown 全文拼接版）。生成脚本：从 sitemap 取 URL → 抓 main 区域 → markdown 化 → 用 `---` 分隔。

**Cloudflare WAF 规则**（Dashboard → Security → WAF → Custom rules）：

```
(http.user_agent contains "GPTBot") or
(http.user_agent contains "ClaudeBot") or
(http.user_agent contains "anthropic-ai") or
(http.user_agent contains "PerplexityBot") or
(http.user_agent contains "Google-Extended") or
(http.user_agent contains "OAI-SearchBot") or
(http.user_agent contains "Bytespider") or
(http.user_agent contains "Applebot-Extended") or
(http.user_agent contains "CCBot")
→ Action: Skip → All remaining custom rules + Bot Fight Mode + Super Bot Fight Mode
```

**两个面板都要查**（Cloudflare 把 AI 爬虫管理拆在两处）：
- **Security → WAF → Custom rules**：上面的规则
- **Security → Bots → AI Crawlers and Scrapers**（部分套餐叫 "AI Audit" 或 "Manage AI bots"）：把 GPTBot / ClaudeBot / PerplexityBot / Google-Extended / Applebot-Extended 全部切到 **Allow**（不是 Block / Challenge）。Cloudflare 在 2024-09 推出过一键 "Block AI bots" 选项，**确认它处于关闭状态**。
- **Security → Bots → Bot Fight Mode / Super Bot Fight Mode**：保持开启可以挡刷站，但要在上一步 WAF 自定义规则里 Skip 掉 AI UAs，否则它会先于自定义规则触发。

**Nginx**（如果直接由 Nginx 控制访问）：

```nginx
# 删除任何形如 if ($http_user_agent ~* "bot|crawl|spider") { return 403; } 的规则
# 或显式放行：
map $http_user_agent $is_ai_bot {
    default 0;
    ~*GPTBot 1;
    ~*ClaudeBot 1;
    ~*anthropic-ai 1;
    ~*PerplexityBot 1;
    ~*Google-Extended 1;
    ~*OAI-SearchBot 1;
    ~*Bytespider 1;
    ~*CCBot 1;
}
# 然后在限速/拦截块前 if ($is_ai_bot) { ... } 放行
```

#### 1.3 SSR / 预渲染

按 Discovery 阶段识别出的技术栈分别处理：
- **WordPress / 传统 PHP CMS**：默认是服务端渲染，重点检查是否有插件（如 WP Rocket、Cache Enabler）的"Lazy Load HTML / Defer JS"误把正文延迟到 JS。也要检查主题是否把核心文本塞进了 `<picture>` 或 SVG。
- **Next.js / Nuxt / SvelteKit** → 关键页改用 `getStaticProps` / App Router server components / `+page.server.ts`，确认 `view-source:` 看到的是正文而非空容器。
- **纯 React / Vue SPA** → 配置 prerender.io / rendertron / Cloudflare Workers 预渲染，根据 UA 给爬虫返回静态 HTML。
- **Webflow / Wix / Squarespace** → 默认 SSR，重点放在 schema 注入和 robots/Cloudflare 层。

#### 1.4 验收（必须跑通）

```bash
for UA in "GPTBot/1.0" "ClaudeBot/1.0" "PerplexityBot/1.0" "Google-Extended" "Bytespider"; do
  CODE=$(curl -s -o /tmp/_body -w "%{http_code}" -A "$UA" https://halofortune.com.au/)
  HITS=$(grep -ciE "mortgage|home loan|broker" /tmp/_body)
  echo "$UA → HTTP $CODE, keyword hits: $HITS"
done
```

✅ 通过条件：所有 UA 返回 **200** 且 keyword hits ≥ 3。

---

### Phase 2 · 实体收敛 + 主题集群 + 结构化数据（2-4 周）

#### 2.1 首页"硬定义"段（放在 `<h1>` 下方第一屏）

英文版（en-AU）：

> **Halo Fortune is a Melbourne-based mortgage broker** (ABN [TODO] · ACL/ACR #[TODO] · MFAA/FBAA member #[TODO]) helping Australian residents and Chinese-speaking buyers secure first-home, investment, owner-occupied and bridging loans across Victoria. Based at 332 Kings Way, South Melbourne. Bilingual service in English and 中文.

中文版（zh-CN）：

> **Halo Fortune 是位于墨尔本的持牌房贷经纪机构**（ABN [TODO] · 澳洲信贷牌照 ACL/ACR #[TODO] · MFAA/FBAA 会员 #[TODO]），为澳洲本地居民及华人买家提供首套房、投资房、自住房和过桥贷款服务，覆盖维州全境。地址：332 Kings Way, South Melbourne。中英双语服务。

#### 2.2 信息架构（新建以下页面，每页 1500-2500 词）

```
/                                         # 首页：实体定义 + 4 大产品入口 + 经纪人 + 评价 + FAQ x6
/mortgage-broker-melbourne/               # Pillar 页（topic hub）
  /first-home-buyer/                      # 子话题
  /investment-property-loan/              # 子话题
  /owner-occupied-loan/                   # 子话题
  /refinance/                             # 子话题
  /bridging-loan/                         # 子话题
/about/wei-chen/                          # 经纪人 bio + 执照 + 专长
/about/                                   # 公司页
/contact/                                 # 联系页（含 LocalBusiness 数据）
/zh/                                      # 中文版镜像（同上结构）
```

每个子话题页内链回 Pillar 页（"← Back to Mortgage Broker Melbourne"）+ 互链 1-2 个相关子话题。

#### 2.3 全站 `<head>` 注入（每页都要有）

**Organization + LocalBusiness JSON-LD**（首页）：

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": ["LocalBusiness", "FinancialService"],
      "additionalType": "https://www.wikidata.org/wiki/Q1378050",
      "@id": "https://halofortune.com.au/#org",
      "name": "Halo Fortune",
      "alternateName": "Halo Fortune Group",
      "legalName": "Halo Fortune Group Pty Ltd",
      "url": "https://halofortune.com.au/",
      "logo": "https://halofortune.com.au/assets/logo.png",
      "image": "https://halofortune.com.au/assets/logo.png",
      "description": "Melbourne-based mortgage broker serving Australian residents and Chinese-speaking buyers with first-home, investment, owner-occupied and bridging loans.",
      "slogan": "Bilingual mortgage broking for Melbourne — English and 中文.",
      "priceRange": "$$",
      "telephone": "+61-3-9999-9768",
      "email": "kevin@halofortune.com",
      "address": {
        "@type": "PostalAddress",
        "streetAddress": "332 Kings Way",
        "addressLocality": "South Melbourne",
        "addressRegion": "VIC",
        "postalCode": "3205",
        "addressCountry": "AU"
      },
      "geo": {"@type": "GeoCoordinates", "latitude": "[TODO]", "longitude": "[TODO]"},
      "openingHoursSpecification": [{
        "@type": "OpeningHoursSpecification",
        "dayOfWeek": ["Monday","Tuesday","Wednesday","Thursday","Friday"],
        "opens": "09:00", "closes": "18:00"
      }],
      "areaServed": [
        {"@type": "AdministrativeArea", "name": "Victoria"},
        {"@type": "City", "name": "Melbourne"}
      ],
      "knowsLanguage": ["en-AU", "zh-Hans", "zh-Hant"],
      "identifier": [
        {"@type": "PropertyValue", "propertyID": "ABN",  "value": "[TODO]"},
        {"@type": "PropertyValue", "propertyID": "ACL",  "value": "[TODO]"},
        {"@type": "PropertyValue", "propertyID": "MFAA", "value": "[TODO]"},
        {"@type": "PropertyValue", "propertyID": "AFCA", "value": "[TODO]"}
      ],
      "hasOfferCatalog": {
        "@type": "OfferCatalog",
        "name": "Home Loan Services",
        "itemListElement": [
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "First Home Buyer Loan", "url": "https://halofortune.com.au/first-home-buyer/"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Investment Property Loan", "url": "https://halofortune.com.au/investment-property-loan/"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Owner-Occupied Loan", "url": "https://halofortune.com.au/owner-occupied-loan/"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Refinance", "url": "https://halofortune.com.au/refinance/"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Bridging Loan", "url": "https://halofortune.com.au/bridging-loan/"}}
        ]
      },
      "sameAs": [
        "https://www.facebook.com/[TODO]",
        "https://www.linkedin.com/company/[TODO]",
        "https://brokerpages.com.au/mortgage-broker/wei-chen/"
      ]
    },
    {
      "@type": "Person",
      "@id": "https://halofortune.com.au/about/wei-chen/#person",
      "name": "Wei Chen",
      "jobTitle": "Mortgage Broker",
      "worksFor": {"@id": "https://halofortune.com.au/#org"},
      "knowsLanguage": ["en", "zh"],
      "hasCredential": [
        {"@type": "EducationalOccupationalCredential", "credentialCategory": "license", "name": "Australian Credit Licence Representative", "identifier": "[TODO]"},
        {"@type": "EducationalOccupationalCredential", "credentialCategory": "membership", "name": "MFAA/FBAA Member", "identifier": "[TODO]"}
      ]
    }
  ]
}
</script>
```

> ⚠ **Schema 类型注意**：早期版本曾用 `"@type": "MortgageBroker"` —— 这**不是** Schema.org 合法类型，Rich Results Test 会报错。改用 `["LocalBusiness", "FinancialService"]` + `additionalType` 指向 Wikidata Q1378050（"mortgage broker"）传达语义。

**FAQPage JSON-LD**（每个产品页底部 8-12 条问答）：

> ⚠ **现实预期校准**：Google 自 2023-08 起**已下线大多数站点的 FAQ 富片段**（仅政府/医疗等权威类还保留）。**FAQPage schema 现在的主要价值在 AI 引擎**——ChatGPT / Perplexity / Google AI Overviews / 文心一言仍然把 FAQPage 视为高优先级抽取目标。所以仍要做，但不要向用户承诺 Google SERP 上的 FAQ 折叠效果。

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {"@type": "Question", "name": "How much can I borrow as a first home buyer in Melbourne?",
     "acceptedAnswer": {"@type": "Answer", "text": "..."}},
    {"@type": "Question", "name": "Do I need permanent residency to get a home loan in Australia?",
     "acceptedAnswer": {"@type": "Answer", "text": "..."}}
    /* ... */
  ]
}
</script>
```

**Article JSON-LD**（每个话题页）：必须含 `author`（指向 Wei Chen `@id`）、`datePublished`、`dateModified`、`publisher`（指向 Org `@id`）。

#### 2.4 多语言 hreflang（每页 `<head>`）

```html
<link rel="alternate" hreflang="en-AU" href="https://halofortune.com.au/first-home-buyer/" />
<link rel="alternate" hreflang="zh-Hans" href="https://halofortune.com.au/zh/first-home-buyer/" />
<link rel="alternate" hreflang="x-default" href="https://halofortune.com.au/first-home-buyer/" />
```

#### 2.5 `<title>` / `<meta description>` 模板

```
<title>{产品名} in Melbourne | Halo Fortune Mortgage Broker</title>
<meta name="description" content="{50-160 字符，含主关键词 + 地理 + 业务标识 + 一个差异点}" />
```

例：`<title>First Home Buyer Loans in Melbourne | Halo Fortune Mortgage Broker</title>`

#### 2.6 验收

- 提交 `/sitemap.xml` 到 Google Search Console + Bing Webmaster + IndexNow
- https://search.google.com/test/rich-results 测试每类 schema：必须 0 error
- 提问 ChatGPT / Perplexity："halofortune.com.au 是什么公司" → 应稳定回答 "Melbourne mortgage broker"，不再混淆为 usehalo.com / halo-technologies.com

---

### Phase 3 · 硬数据 + FAQ + 时间戳（持续，首批 3-4 周）

每个产品页必须包含以下 5 块（缺一块退回重写）：

| 块 | 内容 | 形态 |
|---|---|---|
| 利率 / 费用 | 当前指示性利率区间、broker fee 政策（明确写"零费用 / 由贷方支付佣金"） | 表格 + 段落 |
| 准入条件 | 最低收入、最低首付、LVR、信用要求、签证要求 | 项目符号列表 |
| 流程时间表 | enquiry → pre-approval → settlement 的天数区间 | 编号步骤（带 schema:HowTo 更佳） |
| 合作贷方 | 30+ lenders 的 logo 墙 + 文字列表（CBA / NAB / Westpac / ANZ / Macquarie / ING / Bankwest / ...） | logo grid + 文字 |
| 案例 | 3-5 条 case study：金额、产品、地区、用时、客户类型（脱敏） | 卡片 + 引言 |

**FAQ 必出题（中英各 8-12 条/页，与 schema 对齐）**：

英文：
1. How much can I borrow as a first home buyer in Melbourne?
2. Do I need permanent residency to get a home loan in Australia?
3. What is LMI and how do I avoid it?
4. Fixed vs variable rate — which is better in 2026?
5. How long does pre-approval take?
6. What documents do I need for a home loan application?

中文：
1. 墨尔本买房最少需要多少首付？
2. 持 482 / 485 / 500 签证可以申请房贷吗？
3. LMI 是什么？什么情况下可以免除？
4. 找经纪人和直接找银行有什么区别？需要付费吗？
5. 投资房贷款和自住房贷款有什么不同？
6. 贷款审批一般要多久？
7. 海外收入可以用来计算可贷金额吗？
8. 维州印花税新政（First Home Buyer / 海外买家）目前怎么算？

每条答案 **80-150 词**，给具体数字 + 时间 + 出处链接。每条答案末尾必须能独立成段——AI 引擎抽取时常常只取单条 Q&A，孤立可读才有引用价值。

**时间戳**：
- 每页底部 `Last updated: 2026-05-09`，与 JSON-LD `dateModified` 同步
- 每次内容修改更新此字段（CI 可自动化）

**权威外链**（提升被引用率）：
- RBA 现金利率：https://www.rba.gov.au/statistics/cash-rate/
- APRA serviceability buffer：https://www.apra.gov.au/
- 维州印花税：https://www.sro.vic.gov.au/
- ASIC MoneySmart：https://moneysmart.gov.au/

**E-E-A-T 信号**：
- Wei Chen 个人页：执照、会员号、年限、累计放款、Google Reviews 数量
- 客户评价：拉 Google / ProductReview / Brokerpages 真实评价 + `Review` schema

**澳洲合规免责声明**（在每个产品页 / FAQ 区块下方放固定区块，中英双语）：

> *General advice disclaimer (ASIC RG 234)*: The information on this page is general in nature only and has been prepared without taking into account your personal objectives, financial situation or needs. Before acting on any information you should consider its appropriateness, having regard to your own objectives, financial situation and needs, and seek personal financial advice. Halo Fortune Group Pty Ltd · ABN [TODO] · Australian Credit Licence/Representative #[TODO] · MFAA/FBAA Member #[TODO] · AFCA Member #[TODO].

> *一般建议声明*：本页信息仅为一般性资讯，未考虑您的个人目标、财务状况或需求。在依据本页任何信息行动前，请评估其是否适合您的具体情况，并寻求专业财务建议。Halo Fortune Group Pty Ltd · ABN [TODO] · 澳洲信贷牌照/代表 #[TODO] · MFAA/FBAA 会员 #[TODO] · AFCA 会员 #[TODO]。

#### 验收

- 用 Profound / Otterly / AthenaHQ（任一 GEO 监测工具）持续追踪 30 天
- 在 Perplexity 提问 "current home loan rate Australia 2026"、"first home buyer grant Victoria 2026"、"墨尔本华人房贷经纪推荐" → halofortune.com.au 至少出现在长尾问题的引用源中

---

## 4. 不要做的事（Out of Scope）

- ❌ 不要"顺手"重做视觉设计 / 改前端框架 / 重构后端
- ❌ 不要把其他业务线（教育中介、电商、HR）的流量入口砍掉——降级到页脚或迁子域即可，避免破坏现有客户路径
- ❌ 不要伪造 ABN / 执照号 / 利率 / 案例数据；缺数据保留 `[TODO: 待 Halo Fortune 提供]` 占位并在 PR 描述中列清单
- ❌ 不要为了塞关键词牺牲可读性；AI 引擎对 keyword stuffing 的惩罚比传统 SEO 更狠
- ❌ 不要使用未声明来源的统计数字

---

## 5. 交付物清单

最终向用户提交一个 PR，包含：

1. `/robots.txt` + `/llms.txt` + `/llms-full.txt`
2. `/sitemap.xml`（含所有新建话题页 + 中文版镜像）
3. WAF / 服务器配置改动说明（写在 PR 描述里，因为这部分通常不在代码仓库）
4. 首页 + 6 个话题页 + 经纪人页 + 中文版的内容与 schema
5. 全站 footer：ABN / ACL / MFAA / AFCA 编号显示（按合规要求）
6. 每个产品页 / FAQ 页：ASIC RG 234 general advice 免责声明（中英双语）
7. `docs/geo-discovery-notes.md`（Phase 0 的发现）
8. `docs/geo-implementation-log.md`（每个 Phase 的完成情况、跳过项、待用户提供的字段清单）
9. PR 描述含验收命令的实际输出（curl 矩阵 + Rich Results Test 通过截图）

**示例 sitemap.xml**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:xhtml="http://www.w3.org/1999/xhtml">
  <url>
    <loc>https://halofortune.com.au/</loc>
    <lastmod>2026-05-09</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
    <xhtml:link rel="alternate" hreflang="en-AU" href="https://halofortune.com.au/"/>
    <xhtml:link rel="alternate" hreflang="zh-Hans" href="https://halofortune.com.au/zh/"/>
    <xhtml:link rel="alternate" hreflang="x-default" href="https://halofortune.com.au/"/>
  </url>
  <url>
    <loc>https://halofortune.com.au/mortgage-broker-melbourne/</loc>
    <lastmod>2026-05-09</lastmod>
    <changefreq>monthly</changefreq>
    <priority>0.9</priority>
  </url>
  <!-- ... first-home-buyer / investment-property-loan / owner-occupied-loan / refinance / bridging-loan / about/wei-chen / contact + 中文镜像 -->
</urlset>
```

---

## 6. 缺信息回报模板

完成 Discovery 后如发现需要用户提供信息，按下列模板列出：

```markdown
### 待用户提供的信息（阻塞 Phase 2/3）

- [ ] ABN：________
- [ ] ACL / ACR 编号：________
- [ ] MFAA 或 FBAA 会员编号：________
- [ ] **AFCA 会员编号**（持牌信贷机构强制披露）：________
- [ ] 合作贷方完整名单（30+）
- [ ] Wei Chen 从业年限、累计放款规模
- [ ] 营业时间（每天）
- [ ] 实际可公开的利率区间或"参考利率"政策
- [ ] 5 条脱敏案例（金额 / 产品 / 用时）
- [ ] Google Business Profile 链接、其他社交账号
- [ ] 公司地理坐标（纬度/经度，用 https://www.latlong.net 取）
- [ ] 中文版是否单独子域（zh.halofortune.com.au）或子目录（/zh/）的偏好
- [ ] 教育中介 / 中国市场拓展 / 电商 / HR 业务线**目前是否仍在运营**（Discovery 阶段需向用户确认，影响是否真的要"降级"）
```

---

## 7. 上下文锚点（你可以引用的事实出处）

- 公司目录页：https://www.educationagentsguide.com/australia/halo_fortune_group_melbourne_education_agents.htm
- 经纪人页：https://brokerpages.com.au/mortgage-broker/wei-chen/
- 完整审计原文：`docs/geo-audit-2026-05-09.md`（同仓库）

---

**如对本简报任何条目有歧义，停下来向用户提问，不要凭猜测推进。**
