# GEO 实施交接简报 v2 · halofortune.com.au + haloloan.com.au

> **致接手 Agent**：本文档独立自洽，无需任何先前对话上下文。读完即可开工。
> **本版本（v2）取代 v1**——v1 假设单一域名，与实际架构不符。本版基于多轮审查发现的双域结构（`halofortune.com.au` 集团/英文站 + `haloloan.com.au` 中文房贷站）重写。

---

## 0. Mission

把 Halo Fortune Group 在两个域名上的房贷业务，从"对生成式搜索引擎隐身"的状态，改造成 ChatGPT / Perplexity / Google AI Overviews（英文渠道）+ 文心一言 / Kimi / 豆包 / DeepSeek（中文渠道）在被问及"Melbourne mortgage broker / 墨尔本华人房贷经纪"时**会主动引用的权威源**。

业务焦点（不要发散）：**贷款 / 房贷经纪（Mortgage Broker）**。

---

## 1. 当前真实架构（多轮审查得出）

| 域名 | 默认语言 | Google 索引 | 角色（需定型） |
|---|---|---|---|
| `halofortune.com.au` | 英文 | `site:` 返 0 条；403 全站 | **集团 + 英文房贷站** |
| `halofortune.com.au/zh` | 中文（新建） | 0 条；403 | **删除**——与 haloloan 冲突 |
| `haloloan.com.au` | 中文（已有） | 首页可见，标题 "Home - 墨尔本最权威的房屋贷款公司" | **中文房贷正典（canonical）** |
| `haloloan.com.au/zh` | 推断中文 | 不显形 | **删除**——haloloan 已是中文 default，`/zh` 冗余 |

**所属信息**：
- 公司：Halo Fortune Group Pty Ltd
- 地址：332 Kings Way, South Melbourne, VIC 3205
- 集团电话：+61 3 9999 9768；邮箱：kevin@halofortune.com（集团）/ weiming@halofortune.com.au（房贷负责人，已在 haloloan 隐私政策 PDF 公开）
- Director / 房贷负责人：Wei Chen（基于 Brokerpages 显示工作地点 Camberwell, VIC）
- 中文品牌名：**环澳信贷**（Halo Loan）
- 业务范围：First Home Buyer · Property Investor · Owner-Occupied · Bridging Loan · Commercial · Vehicle · Personal/Private Loan
- 已知页面（haloloan.com.au）：首页、`/个人私人贷款/`、隐私政策 PDF `/resource/Halo_Loan_Privacy_Policy.pdf`

---

## 2. 战略决策（已锁定，不再讨论）

```
halofortune.com.au           ←  集团 + 英文房贷站（en-AU）
└─ /                            首页：集团身份 + 英文房贷主入口
└─ /mortgage-broker-melbourne/  pillar
   ├─ /first-home-buyer/
   ├─ /investment-property-loan/
   ├─ /owner-occupied-loan/
   ├─ /refinance/
   └─ /bridging-loan/
└─ /about/wei-chen/
└─ /about/  /contact/
└─ /zh/                         ❌ 删除，301 → haloloan.com.au 对应页

haloloan.com.au              ←  中文房贷正典（zh-Hans，环澳信贷）
└─ /                            首页：中文身份 + 中文房贷主入口（保留现有索引）
└─ /房屋贷款/  /投资贷款/  /再融资/  /过桥贷款/  /商业贷款/  /个人私人贷款/  
   /首次购房/  /海外人士贷款/    （保留中文字符 URL，已有 SEO 沉淀）
└─ /关于/wei-chen/
└─ /关于/  /联系我们/
└─ /zh                          ❌ 删除（haloloan 已是中文 default）
└─ /en/                         ❓ 可选：英文镜像（如果不想做，跨域 hreflang 指向 halofortune.com.au）
```

**双域跨链规则**：
- 跨域 `hreflang`：`halofortune.com.au` (en-AU) ↔ `haloloan.com.au` (zh-Hans)
- 同一 Organization JSON-LD（`@id` 相同），双域互相 `sameAs`
- 跨域 canonical：每个英文话题页 → 自身；每个中文话题页 → 自身；不要互指 canonical

---

## 3. 用户必须先提供的事实（阻塞 Phase 4 起所有工作）

```markdown
- [ ] ABN：________
- [ ] ACL（Australian Credit Licence）或 ACR（Credit Representative）编号：________
- [ ] MFAA 或 FBAA 会员编号：________
- [ ] AFCA 会员编号（持牌信贷机构强制披露）：________
- [ ] Wei Chen 从业年限、累计放款规模（用于 Person 页 + bio）
- [ ] 营业时间（每天，含周末）
- [ ] 公司精确地理坐标（lat/lng）
- [ ] 合作贷方完整名单（30+，至少 10 个 logo 可公开使用）
- [ ] 实际可公开的利率区间或"参考利率"政策（或明确写"按贷方实时报价"）
- [ ] 5 条脱敏案例（金额、产品、地区、用时、客户类型）
- [ ] Google Business Profile 链接、其他社交账号
- [ ] 是否要在 haloloan.com.au 加 `/en/` 英文镜像（默认：不加，靠 halofortune 承接英文）
```

**未提供的字段全部留 `[TODO]` 占位，禁止编造**。每个 `[TODO]` 必须出现在 PR 描述的"待用户提供"清单里。

---

## 4. 执行计划（按依赖顺序，不可乱）

### Phase 0 · Discovery（30 分钟，两域都做）

把结果写到 `docs/geo-discovery-notes.md` 后再开始动手改。

```bash
DOMAINS=("halofortune.com.au" "haloloan.com.au")

for D in "${DOMAINS[@]}"; do
  echo "============ $D ============"
  for UA in "" "Mozilla/5.0" "GPTBot/1.0" "ClaudeBot/1.0" "PerplexityBot/1.0" \
            "Google-Extended" "OAI-SearchBot/1.0" "Bytespider" "Baiduspider/2.0" \
            "Sogou web spider"; do
    printf "  %-22s → " "${UA:-empty}"
    curl -sIL -A "$UA" --max-time 10 "https://$D/" | head -1
  done
  echo "  --- robots.txt ---"
  curl -s "https://$D/robots.txt" | head -30
  echo "  --- sitemap.xml ---"
  curl -s "https://$D/sitemap.xml" | head -20
  echo "  --- llms.txt ---"
  curl -s "https://$D/llms.txt" | head -10
  echo "  --- HTML keyword check ---"
  curl -s -A "ClaudeBot/1.0" "https://$D/" | grep -ciE "mortgage|home loan|broker|首付|房贷|贷款"
  echo "  --- structured data ---"
  curl -s "https://$D/" | grep -oE 'application/ld\+json[^<]{0,200}' | head -3
  echo "  --- tech stack hints ---"
  curl -sI "https://$D/" | grep -iE "server|x-powered-by|cf-ray|x-cache"
done
```

**清单产出**：
- 当前哪些 UA 被 403、哪些 200（两域都要表）
- 各域的 robots.txt / sitemap.xml / llms.txt 状态
- 是否 SSR
- CMS / 框架 / Cloudflare 套餐
- 已有结构化数据类型（如有）
- 已有页面 URL 列表（特别是 haloloan 的中文字符 URL）

> ⚠ **没有 shell 怎么办**：(a) https://www.webpagetest.org/ 自定义 UA；(b) Chrome DevTools → Network → UA override；(c) `wget --user-agent=...`；(d) https://search.google.com/test/mobile-friendly 看渲染后 HTML。

> 🛑 **Discovery 完成后停下来汇报**——把笔记 push 到当前分支，等待用户/审查 Agent 确认后再进 Phase 1。

---

### Phase 1 · 解除爬虫拦截（**双域同时做**，1 周内，阻塞所有后续）

#### 1.1 `robots.txt`（两个域各放一份，根目录）

英文站 `halofortune.com.au/robots.txt`：

```txt
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /wp-admin/
Disallow: /api/
Disallow: /zh                  # 旧 /zh 路径已 301，但仍显式禁止索引

# Generative AI crawlers — explicit allow（每个 UA 块在 robots 协议中独立）
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

User-agent: Amazonbot
Allow: /

User-agent: cohere-ai
Allow: /

User-agent: Diffbot
Allow: /

User-agent: YouBot
Allow: /

Sitemap: https://halofortune.com.au/sitemap.xml
```

中文站 `haloloan.com.au/robots.txt`（**额外放行中文 AI 爬虫**）：

```txt
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /wp-admin/
Disallow: /zh                  # 旧 /zh 路径已 301，禁止索引
Disallow: /resource/*.pdf$     # 禁止把 PDF 当主索引（HTML 版才是正版）

# Generative AI crawlers（英文 + 中文渠道全覆盖）
User-agent: GPTBot
Allow: /

User-agent: ClaudeBot
Allow: /

User-agent: PerplexityBot
Allow: /

User-agent: Google-Extended
Allow: /

User-agent: Bytespider
Allow: /

User-agent: Baiduspider
Allow: /

User-agent: Baiduspider-render
Allow: /

User-agent: Sogou web spider
Allow: /

User-agent: YisouSpider
Allow: /

User-agent: 360Spider
Allow: /

User-agent: HaoSouSpider
Allow: /

User-agent: Yeti
Allow: /

User-agent: NaverBot
Allow: /

Sitemap: https://haloloan.com.au/sitemap.xml
```

#### 1.2 新增 `/llms.txt` 与 `/llms-full.txt`（两域各一份，仿 https://llmstxt.org）

英文站 `halofortune.com.au/llms.txt`：

```txt
# Halo Fortune

> Melbourne-based mortgage broker (ABN [TODO] · ACL/ACR #[TODO]) helping Australian residents and Chinese-speaking buyers with first-home, investment, owner-occupied and bridging loans. English-language site of Halo Fortune Group; Chinese site at https://haloloan.com.au.

## Core Services

- [Mortgage Broker Melbourne (Pillar)](https://halofortune.com.au/mortgage-broker-melbourne/): Full topic hub.
- [First Home Buyer Loans](https://halofortune.com.au/first-home-buyer/): FHOG, deposit, LMI.
- [Investment Property Loans](https://halofortune.com.au/investment-property-loan/): LVR, serviceability, rental income.
- [Owner-Occupied Loans](https://halofortune.com.au/owner-occupied-loan/): Fixed vs variable, offset vs redraw.
- [Refinance](https://halofortune.com.au/refinance/): When and how.
- [Bridging Loans](https://halofortune.com.au/bridging-loan/): Buy-before-sell scenarios.

## About

- [Wei Chen — Director & Mortgage Broker](https://halofortune.com.au/about/wei-chen/)
- [Contact](https://halofortune.com.au/contact/): 332 Kings Way, South Melbourne · +61 3 9999 9768

## 中文 / Chinese

Chinese-language service is provided under our 环澳信贷 (Halo Loan) brand: https://haloloan.com.au/
```

中文站 `haloloan.com.au/llms.txt`：

```txt
# 环澳信贷 Halo Loan

> 墨尔本华人房贷经纪机构（ABN [TODO] · 澳洲信贷牌照 #[TODO] · MFAA/FBAA #[TODO] · AFCA #[TODO]），为澳洲华人买家提供首套房、投资房、自住房、过桥贷款、商业贷款及个人贷款服务。隶属 Halo Fortune Group Pty Ltd。英文站点：https://halofortune.com.au。

## 核心服务

- [房屋贷款](https://haloloan.com.au/房屋贷款/)
- [投资贷款](https://haloloan.com.au/投资贷款/)
- [再融资贷款](https://haloloan.com.au/再融资/)
- [过桥贷款](https://haloloan.com.au/过桥贷款/)
- [商业贷款](https://haloloan.com.au/商业贷款/)
- [个人/私人贷款](https://haloloan.com.au/个人私人贷款/)
- [首次购房](https://haloloan.com.au/首次购房/)
- [海外人士贷款](https://haloloan.com.au/海外人士贷款/)

## 关于

- [关于 Wei Chen](https://haloloan.com.au/关于/wei-chen/)
- [联系我们](https://haloloan.com.au/联系我们/)：墨尔本 South Melbourne · +61 3 9999 9768

## English

For English service, visit our parent group site: https://halofortune.com.au/
```

`/llms-full.txt`：脚本化生成——从 sitemap 取所有 URL → 抓 `<main>` → markdown 化 → 用 `---` 分隔。

#### 1.3 WAF / Cloudflare 放行（两个域，**两个面板都要查**）

| 面板 | 检查 / 设置 |
|---|---|
| Security → Bots → Configure Super Bot Fight Mode | "Definitely automated" 改为 **Allow** 或 **Managed Challenge**（不要 Block） |
| Security → Bots → AI Crawlers and Scrapers（"AI Audit" 或 "Manage AI bots"） | GPTBot / ClaudeBot / PerplexityBot / Google-Extended / Applebot-Extended / Bytespider / Baiduspider 全部 **Allow** |
| Security → WAF → Custom rules | 删除任何 `cf.client.bot` 或 `http.user_agent contains "bot"` 全局拦截规则；如保留风控规则，必须在前面加一条放行 AI UA 的 Skip 规则 |
| Security → WAF → Managed Rules → OWASP | 临时关闭跑一次 curl 矩阵，确认是否是 OWASP 在拦 |

**Cloudflare 放行规则模板**（在 Custom rules 顶部）：

```
(http.user_agent contains "GPTBot") or
(http.user_agent contains "OAI-SearchBot") or
(http.user_agent contains "ChatGPT-User") or
(http.user_agent contains "ClaudeBot") or
(http.user_agent contains "anthropic-ai") or
(http.user_agent contains "Claude-Web") or
(http.user_agent contains "PerplexityBot") or
(http.user_agent contains "Perplexity-User") or
(http.user_agent contains "Google-Extended") or
(http.user_agent contains "Applebot-Extended") or
(http.user_agent contains "CCBot") or
(http.user_agent contains "Amazonbot") or
(http.user_agent contains "Bytespider") or
(http.user_agent contains "Baiduspider") or
(http.user_agent contains "Sogou web spider") or
(http.user_agent contains "YisouSpider")
→ Action: Skip → 跳过 All remaining custom rules + Bot Fight Mode + Super Bot Fight Mode
```

#### 1.4 SSR / 预渲染

按 Discovery 识别的栈处理：
- **WordPress / 传统 PHP CMS**：默认 SSR，重点检查缓存插件（WP Rocket、Cache Enabler、LiteSpeed）的"Lazy Load HTML / Defer JS"是否把正文延迟。也要查主题是否把核心文本塞进 `<picture>` / SVG / `<canvas>`。
- **Next.js / Nuxt / SvelteKit**：关键页用 SSG / SSR / server components，确认 `view-source:` 看到的是正文不是空容器。
- **纯 React / Vue SPA**：配 prerender.io / rendertron / Cloudflare Workers 预渲染，UA 命中爬虫返静态 HTML。
- **Webflow / Wix / Squarespace**：默认 SSR，重点放 schema 注入和 robots/Cloudflare 层。

#### 1.5 验收（必须全过）

```bash
for D in halofortune.com.au haloloan.com.au; do
  echo "=== $D ==="
  for UA in "GPTBot/1.0" "ClaudeBot/1.0" "PerplexityBot/1.0" "Google-Extended" "Bytespider" "Baiduspider/2.0"; do
    CODE=$(curl -s -o /tmp/_body -w "%{http_code}" -A "$UA" "https://$D/")
    HITS=$(grep -ciE "mortgage|home loan|broker|贷款|房贷|首付" /tmp/_body)
    echo "  $UA → HTTP $CODE, keyword hits: $HITS"
  done
  curl -s -o /dev/null -w "robots.txt → %{http_code}\n" "https://$D/robots.txt"
  curl -s -o /dev/null -w "llms.txt → %{http_code}\n" "https://$D/llms.txt"
  curl -s -o /dev/null -w "sitemap.xml → %{http_code}\n" "https://$D/sitemap.xml"
done
```

✅ 通过条件：所有 UA / 所有特殊文件 → **HTTP 200**，且 keyword hits ≥ **3**（halofortune 用英文关键词，haloloan 用中文关键词）。

---

### Phase 2 · 域名收敛与 301 重定向（Phase 1 通过后立即做）

#### 2.1 `/zh` 路径冗余清理

| 旧 URL | 新 URL（301 永久重定向） |
|---|---|
| `https://halofortune.com.au/zh` | `https://haloloan.com.au/` |
| `https://halofortune.com.au/zh/*` | `https://haloloan.com.au/$1`（按对应内容映射；找不到对应就回首页） |
| `https://haloloan.com.au/zh` | `https://haloloan.com.au/` |
| `https://haloloan.com.au/zh/*` | `https://haloloan.com.au/$1` |

Apache `.htaccess` 示例（haloloan）：

```apache
RewriteEngine On
RewriteRule ^zh/?$ / [R=301,L]
RewriteRule ^zh/(.*)$ /$1 [R=301,L]
```

Nginx 示例（halofortune）：

```nginx
location ~ ^/zh/?(.*)$ {
    return 301 https://haloloan.com.au/$1;
}
```

#### 2.2 跨域 `<head>` 注入（每页都要）

英文页（halofortune.com.au）：

```html
<link rel="canonical" href="https://halofortune.com.au/{当前路径}" />
<link rel="alternate" hreflang="en-AU" href="https://halofortune.com.au/{当前路径}" />
<link rel="alternate" hreflang="zh-Hans" href="https://haloloan.com.au/{对应中文页路径}" />
<link rel="alternate" hreflang="x-default" href="https://halofortune.com.au/{当前路径}" />
```

中文页（haloloan.com.au）：

```html
<link rel="canonical" href="https://haloloan.com.au/{当前路径}" />
<link rel="alternate" hreflang="zh-Hans" href="https://haloloan.com.au/{当前路径}" />
<link rel="alternate" hreflang="en-AU" href="https://halofortune.com.au/{对应英文页路径}" />
<link rel="alternate" hreflang="x-default" href="https://halofortune.com.au/{对应英文页路径}" />
```

> 跨域 hreflang 必须**双向**指向，否则 Google 会忽略。

> haloloan 的中文字符 URL 在 hreflang 里要用 **percent-encoded** 形式（即 `/%E6%88%BF%E5%B1%8B%E8%B4%B7%E6%AC%BE/`），不要直接用中文。

---

### Phase 3 · 合规修复（**法律风险，必须最先做**）

#### 3.1 删除"最权威"等绝对最高级表述（haloloan.com.au 当前首页 `<title>` 是 "墨尔本最权威的房屋贷款公司"）

**问题**：
- **ACCC / 澳洲消费者法 §18**：禁止 misleading or deceptive 表述。"最权威"无客观依据，可被认定 misleading conduct，最高罚款 $50M（公司）/ $2.5M（个人）。
- **ASIC RG 234**：金融服务广告中的绝对最高级必须能客观证明且不误导。
- **GEO 风险**：ChatGPT / Perplexity / Google AI Overviews 对"最佳/最权威/第一"等超绝词有降权倾向；同时被 AI 引用时这种自夸 title 会被砍掉，等于浪费关键词位。

**全站搜索并替换**（中英文都查）：

| 禁用词 | 替换示范 |
|---|---|
| 最权威 / 最专业 / 最好 / 第一 / No.1 / 行业第一 | 删除，或改为可证明的事实（如"自 [年份] 服务墨尔本华人客群"） |
| best / leading / #1 / top / most authoritative | 删除，或改为 "specialised in" / "established in 20XX" / "serving [区域]" |
| 保证 / 保证最低利率 / guaranteed lowest rate | 全删——这在 ASIC RG 234 下属违规 |

**首页 title 改写示范**：

中文：`环澳信贷 Halo Loan | 墨尔本华人房贷经纪 · 自住房 / 投资房 / 商业贷款`

英文：`Halo Fortune | Melbourne Mortgage Broker · First-Home, Investment, Refinance & Bridging Loans`

#### 3.2 全站底部添加合规披露区块（强制）

中文站（haloloan.com.au）每页底部：

```html
<footer class="compliance-disclosure">
  <p>环澳信贷 Halo Loan 是 Halo Fortune Group Pty Ltd 的中文房贷服务品牌。</p>
  <p>ABN [TODO] · 澳洲信贷牌照/代表 #[TODO] · MFAA/FBAA 会员 #[TODO] · AFCA 会员 #[TODO]。</p>
  <p><strong>一般建议声明（ASIC RG 234）</strong>：本网站信息仅为一般性资讯，未考虑您的个人目标、财务状况或需求。在依据本网站任何信息行动前，请评估其是否适合您的具体情况，并寻求专业财务建议。</p>
</footer>
```

英文站（halofortune.com.au）每页底部：

```html
<footer class="compliance-disclosure">
  <p>Halo Fortune Group Pty Ltd · ABN [TODO] · Australian Credit Licence/Representative #[TODO] · MFAA/FBAA Member #[TODO] · AFCA Member #[TODO].</p>
  <p><strong>General advice disclaimer (ASIC RG 234)</strong>: The information on this website is general in nature only and has been prepared without taking into account your personal objectives, financial situation or needs. Before acting on any information, you should consider its appropriateness, having regard to your own objectives, financial situation and needs, and seek personal financial advice.</p>
</footer>
```

#### 3.3 隐私政策从 PDF 改为 HTML（haloloan）

当前 `/resource/Halo_Loan_Privacy_Policy.pdf` 是 PDF——AI 引擎对 PDF 解析能力差，且当前 robots.txt 已禁止索引 PDF。

新建 `/隐私政策/`（HTML 版），保留 PDF 作为可下载副本。HTML 版必须包含：
- 信用报告披露（Privacy Act 1988, Part IIIA）
- 客户信息使用范围
- 第三方共享（贷方、信用报告机构）
- 联系/投诉渠道（含 AFCA 与 OAIC）

---

### Phase 4 · 跨域 Schema.org 图谱

#### 4.1 全站根 JSON-LD（两域使用同一 `@id`，互相 `sameAs`）

放在两个站每页 `<head>` 里：

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": ["LocalBusiness", "FinancialService"],
      "additionalType": "https://www.wikidata.org/wiki/Q1378050",
      "@id": "https://halofortune.com.au/#org",
      "name": "Halo Fortune Group",
      "alternateName": ["Halo Loan", "环澳信贷"],
      "legalName": "Halo Fortune Group Pty Ltd",
      "url": "https://halofortune.com.au/",
      "logo": "https://halofortune.com.au/assets/logo.png",
      "image": "https://halofortune.com.au/assets/logo.png",
      "description": "Melbourne-based mortgage broker serving Australian residents and Chinese-speaking buyers with first-home, investment, owner-occupied, bridging, commercial and personal loans. English brand 'Halo Fortune' at halofortune.com.au; Chinese brand '环澳信贷 Halo Loan' at haloloan.com.au.",
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
        "name": "Loan Services",
        "itemListElement": [
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "First Home Buyer Loan"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Investment Property Loan"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Owner-Occupied Loan"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Refinance"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Bridging Loan"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Commercial Loan"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Vehicle Loan"}},
          {"@type": "Offer", "itemOffered": {"@type": "Service", "name": "Personal/Private Loan"}}
        ]
      },
      "sameAs": [
        "https://halofortune.com.au/",
        "https://haloloan.com.au/",
        "https://www.facebook.com/[TODO]",
        "https://au.linkedin.com/in/wei-chen-43862590",
        "https://brokerpages.com.au/mortgage-broker/wei-chen/"
      ]
    },
    {
      "@type": "Person",
      "@id": "https://halofortune.com.au/about/wei-chen/#person",
      "name": "Wei Chen",
      "jobTitle": "Director & Mortgage Broker",
      "worksFor": {"@id": "https://halofortune.com.au/#org"},
      "knowsLanguage": ["en", "zh-Hans"],
      "hasCredential": [
        {"@type": "EducationalOccupationalCredential", "credentialCategory": "license", "name": "Australian Credit Licence Representative", "identifier": "[TODO]"},
        {"@type": "EducationalOccupationalCredential", "credentialCategory": "membership", "name": "MFAA/FBAA Member", "identifier": "[TODO]"}
      ],
      "sameAs": [
        "https://au.linkedin.com/in/wei-chen-43862590",
        "https://brokerpages.com.au/mortgage-broker/wei-chen/"
      ]
    },
    {
      "@type": "WebSite",
      "@id": "https://halofortune.com.au/#website",
      "url": "https://halofortune.com.au/",
      "name": "Halo Fortune",
      "inLanguage": "en-AU",
      "publisher": {"@id": "https://halofortune.com.au/#org"}
    },
    {
      "@type": "WebSite",
      "@id": "https://haloloan.com.au/#website",
      "url": "https://haloloan.com.au/",
      "name": "环澳信贷 Halo Loan",
      "inLanguage": "zh-Hans",
      "publisher": {"@id": "https://halofortune.com.au/#org"}
    }
  ]
}
</script>
```

> ⚠ **绝对不要写** `"@type": "MortgageBroker"`——这**不是** Schema.org 合法类型，Rich Results Test 会报错。`additionalType` 用 Wikidata Q1378050 表达"房贷经纪"语义。

#### 4.2 每个产品页（中英对应）的 schema 增强

- **Article**：`author` 指向 Wei Chen `@id`，`datePublished` + `dateModified`，`publisher` 指向 Org `@id`，`inLanguage` 对应
- **FAQPage**（每页 8-12 条 Q&A）：见 Phase 6
- **BreadcrumbList**：所有非首页

> ⚠ **校准 Google FAQ 富片段预期**：Google 自 2023-08 起已下线大多数站点的 FAQ 富片段。FAQPage schema 现在的主要价值在 **AI 引擎**（ChatGPT / Perplexity / AI Overviews / 文心一言 / Kimi）——它们仍把 FAQPage 视为高优先级抽取目标。所以仍要做，但不要向客户承诺 Google SERP 上的 FAQ 折叠展示。

---

### Phase 5 · 内容 / 主题集群（按域分头执行）

#### 5.1 halofortune.com.au（英文）

新建 6 个话题页（每页 1500-2500 词），围绕 pillar `/mortgage-broker-melbourne/`：

| URL | 核心覆盖 |
|---|---|
| `/mortgage-broker-melbourne/` | Pillar：服务全景 + 链向所有子话题 |
| `/first-home-buyer/` | FHOG VIC、印花税豁免、deposit、LMI、Home Guarantee Scheme |
| `/investment-property-loan/` | LVR、serviceability、租金计入、negative gearing |
| `/owner-occupied-loan/` | 固定 vs 浮动、offset vs redraw、repayment 类型 |
| `/refinance/` | 何时 refinance、cash-out、cashback、loyalty tax |
| `/bridging-loan/` | Buy-before-sell、peak debt、利息累积 |

每页结构（强制）：
1. H1（含主关键词 + Melbourne）
2. 60-80 词硬定义段
3. **5 类硬数据块**（见 Phase 6）
4. 流程时间表（带 HowTo schema）
5. 8-12 条 FAQ + FAQPage schema
6. 内链回 pillar + 互链 1-2 个相关子话题
7. CTA + 合规免责声明

#### 5.2 haloloan.com.au（中文）

**保留所有现有中文字符 URL**（已有 SEO 沉淀）。在现有页面基础上：
- 注入 schema（Article / FAQPage / BreadcrumbList）
- 补 8-12 条 FAQ（中文）
- 加 5 类硬数据块
- 加合规免责声明
- 加 hreflang 到 halofortune 对应页

如果某个英文页在中文站没对应（如 `/refinance/` ↔ `/再融资/`），新建中文版。

**新增页面建议**（针对华人客群高频需求）：
- `/海外人士贷款/` — non-resident lending（含签证 482/485/500/186 各自的 LVR 政策）
- `/首次购房/` — FHOG 中文版（专门解释维州印花税新政）
- `/海外收入认定/` — 海外工资 / 租金 / 商业收入如何计入 serviceability

---

### Phase 6 · 硬数据 + FAQ + 时间戳（持续，首批 3-4 周）

#### 6.1 每个产品页必备 5 类数据块（缺一块退回重写）

| 块 | 内容 | 形态 |
|---|---|---|
| 利率/费用 | 当前指示性利率区间、broker fee 政策（明确写"零费用 / 由贷方支付佣金"） | 表格 + 段落 |
| 准入条件 | 最低收入、最低首付、LVR、信用要求、签证要求 | 项目符号列表 |
| 流程时间表 | enquiry → pre-approval → settlement 的天数区间 | 编号步骤 + HowTo schema |
| 合作贷方 | 30+ lenders 的 logo 墙 + 文字列表 | logo grid + 文字 |
| 案例 | 3-5 条脱敏 case study：金额、产品、地区、用时、客户类型 | 卡片 + 引言 |

#### 6.2 FAQ 必出题（每页中英各 8-12 条）

英文（halofortune）：
1. How much can I borrow as a first home buyer in Melbourne?
2. Do I need permanent residency to get a home loan in Australia?
3. What is LMI and how do I avoid it?
4. Fixed vs variable rate — which is better in 2026?
5. How long does pre-approval take?
6. What documents do I need for a home loan application?
7. Can foreign income be used for serviceability?
8. What is the FHOG in Victoria and who qualifies?

中文（haloloan）：
1. 墨尔本买房最少需要多少首付？
2. 持 482 / 485 / 500 / 186 签证可以申请房贷吗？
3. LMI 是什么？什么情况下可以免除？
4. 找经纪人和直接找银行有什么区别？需要付费吗？
5. 投资房贷款和自住房贷款有什么不同？
6. 贷款审批一般要多久？
7. 海外收入可以用来计算可贷金额吗？
8. 维州印花税新政（First Home Buyer / 海外买家）目前怎么算？
9. 借款人是公司或信托结构有什么注意事项？
10. 能否用现房 equity 做投资房首付？

每条答案 **80-150 词**，给具体数字 + 时间 + 权威出处链接。每条答案末尾必须能独立成段——AI 引擎抽取时常常只取单条 Q&A，孤立可读才有引用价值。

#### 6.3 时间戳

- 每页底部 `Last updated: YYYY-MM-DD`，与 JSON-LD `dateModified` 同步
- 每次内容修改更新此字段（CI 可自动化：modify time → write back）

#### 6.4 权威外链

- RBA 现金利率：https://www.rba.gov.au/statistics/cash-rate/
- APRA serviceability buffer：https://www.apra.gov.au/
- 维州印花税 SRO：https://www.sro.vic.gov.au/
- ASIC MoneySmart：https://moneysmart.gov.au/
- AFCA（投诉机构）：https://www.afca.org.au/
- OAIC（隐私）：https://www.oaic.gov.au/

#### 6.5 E-E-A-T 信号

- Wei Chen 个人页：执照号、MFAA/FBAA 会员号、年限、累计放款规模、专长、Google Reviews 数
- 客户评价：拉 Google Business Profile / ProductReview / Brokerpages 真实评价 + `Review` schema（**只能用真实评价**，禁止伪造）

---

## 5. Sitemap 模板

每域一份。haloloan 的中文 URL 需 percent-encode：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:xhtml="http://www.w3.org/1999/xhtml">
  <url>
    <loc>https://haloloan.com.au/</loc>
    <lastmod>2026-05-09</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
    <xhtml:link rel="alternate" hreflang="zh-Hans" href="https://haloloan.com.au/"/>
    <xhtml:link rel="alternate" hreflang="en-AU" href="https://halofortune.com.au/"/>
  </url>
  <url>
    <loc>https://haloloan.com.au/%E6%88%BF%E5%B1%8B%E8%B4%B7%E6%AC%BE/</loc>
    <lastmod>2026-05-09</lastmod>
    <priority>0.9</priority>
  </url>
  <!-- 其余中文页同上，所有中文路径 percent-encode -->
</urlset>
```

提交到 Google Search Console + Bing Webmaster + IndexNow（两域分别提交）。

---

## 6. 不要做的事（Out of Scope）

- ❌ 不要"顺手"重做视觉设计 / 改前端框架 / 重构后端
- ❌ 不要把 halofortune.com.au 上的非房贷业务（教育中介、Sino Talent、Adcess、Happy Mall、Halo HR）的入口砍掉——降级到页脚或迁子域即可
- ❌ 不要把 haloloan.com.au 的中文字符 URL 改成 ASCII slug——会丢失现有 SEO 沉淀；如有需要，**补充** ASCII alias 而不是替换
- ❌ 不要伪造 ABN / 执照号 / 利率 / 案例数据 / 客户评价；缺数据保留 `[TODO]` 占位
- ❌ 不要使用"最权威 / 最专业 / 第一 / 保证最低利率"等绝对最高级（合规违规）
- ❌ 不要为了塞关键词牺牲可读性
- ❌ 不要写 `"@type": "MortgageBroker"`（非合法类型）
- ❌ 不要把两个域互相 canonical（应各自 canonical 自身页面 + hreflang 跨域指向对方）

---

## 7. 验收（每个 Phase 都要跑通）

### Phase 1 验收
两域 + 所有 AI UA → HTTP 200，HTML 含关键词 ≥ 3。

### Phase 2 验收
- 旧 `/zh` URL → 301 → 新地址（用 `curl -I` 确认 status 301 + Location header）
- 跨域 hreflang 双向（用 https://www.aleydasolis.com/english/international-seo-tools/hreflang-tags-generator/ 校验）

### Phase 3 验收
- 全站 grep 不再有"最权威 / 最专业 / 最好 / 第一 / 保证 / best / leading / #1 / guaranteed"
- 每页底部可见 ABN / ACL / MFAA/FBAA / AFCA + RG 234 disclaimer
- HTML 隐私政策上线（haloloan）

### Phase 4 验收
- https://search.google.com/test/rich-results 测试两域首页 + 任一产品页：**0 error**
- https://validator.schema.org/ 测试通过
- 跨域 `@id` 引用一致

### Phase 5/6 验收
- 在 ChatGPT / Perplexity / Google AI Overviews 提问 "Mortgage broker in South Melbourne for first home buyers" → halofortune.com.au 出现在引用列表
- 在文心一言 / Kimi / 豆包 提问"墨尔本华人房贷经纪"或"墨尔本 mortgage broker 中文" → haloloan.com.au 出现
- Profound / Otterly / AthenaHQ 等 GEO 监测工具持续追踪 30 天后曝光率提升

---

## 8. 交付 PR 描述模板

提交到分支 `claude/audit-website-content-HSS3C` 的 PR 必须包含：

```markdown
## Summary
- Implemented dual-domain GEO architecture per docs/geo-implementation-brief.md (v2)
- halofortune.com.au = English mortgage + group; haloloan.com.au = Chinese canonical (环澳信贷)
- Unblocked AI crawlers (Phase 1) → schema graph → content cluster

## Changes by phase
- [x] Phase 0 Discovery — see docs/geo-discovery-notes.md
- [x] Phase 1 Crawler unblock — see acceptance output below
- [x] Phase 2 /zh redirect map (4 patterns)
- [x] Phase 3 Compliance — removed "最权威" etc.; added RG 234 footer; HTML privacy policy
- [x] Phase 4 Cross-domain JSON-LD graph (Organization + Person + WebSite)
- [x] Phase 5 New topic cluster on halofortune.com.au; schema injection on haloloan.com.au
- [x] Phase 6 5-block data + 8-12 FAQ + Last updated on each product page

## Acceptance evidence
[paste curl matrix output here]
[paste Rich Results Test screenshots/links]
[paste hreflang validator output]

## TODOs blocking — needs user input
- [ ] ABN
- [ ] ACL/ACR number
- [ ] MFAA/FBAA membership number
- [ ] AFCA member number
- [ ] Geographic coordinates
- [ ] Lender panel (30+)
- [ ] Wei Chen years/cumulative loan volume
- [ ] 5 case studies (anonymised)
- [ ] Whether to add /en/ on haloloan.com.au

## Out of scope (explicitly not changed)
[list]
```

---

## 9. 上下文锚点（你可引用的事实出处）

- 公司目录：https://www.educationagentsguide.com/australia/halo_fortune_group_melbourne_education_agents.htm
- 经纪人页：https://brokerpages.com.au/mortgage-broker/wei-chen/
- LinkedIn：https://au.linkedin.com/in/wei-chen-43862590
- haloloan 隐私政策（PDF，待迁 HTML）：https://www.haloloan.com.au/resource/Halo_Loan_Privacy_Policy.pdf
- 完整审计原文（背景）：`docs/geo-audit-2026-05-09.md`（同仓库）

---

## 10. 总原则

1. **先解锁，再优化**：Phase 1 不通过，所有后续工作零价值
2. **合规先于增长**：Phase 3 合规修复必须在内容大改之前完成（"最权威"留一天就是一天的法律风险）
3. **保留沉淀**：haloloan 的中文索引来之不易，**绝不改 URL 结构**
4. **双域协作而非合并**：两个域各服务一个语言市场，schema 用同一 `@id` 串联
5. **缺数据宁缺勿滥**：所有 `[TODO]` 必须显式列入 PR 描述等用户回填，不得编造
6. **每一改都可测**：每个 Phase 必须有 curl / Rich Results Test / hreflang validator 的实际输出贴在 PR 里

---

**有任何条目歧义，停下来用 AskUserQuestion 向用户提问，不要凭猜测推进。**
