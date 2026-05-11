# SEO/GEO 优化 — Round 2 Followup（2026-05-11 12:35）

**前置文档**：[`SEO-GEO-OPTIMIZATION-2026-05-11.md`](./SEO-GEO-OPTIMIZATION-2026-05-11.md)
**复审完成度**：14/18 = 78% 已交付。本文件覆盖剩下的 4 条 + 1 条新 P0 回归。

---

## 0. 给接手 Claude 的简述

上一轮你已经完成了 14 条修复，**质量非常高** —— Organization + WebPage + Person schema、`@graph` 联合、identifier 含 MFAA/AFCA、memberOf、合规防火墙、AI bot 全覆盖、ftfinance hreflang 收口、`/blog` dedup、locale 308、disclaimer 全到位。

但 4 个具体落地差还在，**其中 1 条是新引入的 P0 回归**。请按下面 §1-§4 顺序修。

§1 是新回归，必须立刻看。

合规防火墙规则不变（halo\* ↔ ftfinance 不能在公开层互相提及）—— 见前置文档 §0、§1、§8。

---

## 1. 🔴 新 P0 — haloloan.com.au/sitemap.xml 完全 timeout（回归）

### 证据

```
$ curl -s -o /dev/null -w "%{http_code}\n" --max-time 15 https://haloloan.com.au/sitemap.xml
000

$ for i in 1 2 3; do
    curl -s -o /dev/null -w "haloloan try $i: %{time_total}s\n" --max-time 15 https://haloloan.com.au/sitemap.xml
  done
haloloan try 1: 15.006852s     ← 三次连击全顶到 max-time
haloloan try 2: 15.006197s
haloloan try 3: 15.004617s
```

对比 halofortune（同一份 `src/app/sitemap.ts` 改造）首次就 0.12s，**修复只对 halofortune brand 生效，haloloan 没覆盖到**。这比修复前更糟 —— 之前 haloloan 在 30s 内能拉出 56KB，现在 15s 三连超时。

### 根因推断

上一轮 ISR 改造大概率加了：
```ts
export const revalidate = 86400;
export const dynamic = 'force-static';   // ← 这一行
```

`dynamic = 'force-static'` 在 brand-aware（按 Host 切 brand）的 sitemap 路由上**有害**：
- Build 时 Next.js 只能为单一 brand 静态生成
- 运行时按 Host 切 brand 的逻辑被 bypass
- haloloan brand 走 fallback 路径，每次都触发完整 DB 查询（haloloan 的博客 ~80 篇，比 halofortune 重得多）

### 修复

定位文件：`src/app/sitemap.ts`

**改成**（去掉 force-static，加 Cache-Control 响应头）：
```ts
// 保留 ISR
export const revalidate = 86400;
// 删掉这一行: export const dynamic = 'force-static';

export default async function sitemap({ host }) {
  const brand = resolveBrand(host);   // 多租户切 brand
  const urls = await getBrandSitemapUrls(brand);
  const xml = buildSitemapXml(urls);

  return new Response(xml, {
    status: 200,
    headers: {
      'Content-Type': 'application/xml; charset=utf-8',
      // 关键：让 Vercel Edge 缓存 1 小时，stale-while-revalidate 1 天
      // 第 1 个请求 MISS 时承担生成成本，后续都命中缓存
      'Cache-Control': 'public, s-maxage=3600, stale-while-revalidate=86400',
    },
  });
}
```

如果当前实现用了 Next 14 `sitemap()` 函数返回数组而不是自己写 Response，需要换成自己 build Response 才能加 Cache-Control。具体取决于 Next 版本和现有结构。

或者 **更保险的方案**：在 build 时把每个 brand 的 sitemap 预渲染成静态文件（不依赖运行时），写到 `public/__sitemaps__/{brand}.xml`，然后用 `middleware.ts` 或 `vercel.json` rewrite 把 `https://haloloan.com.au/sitemap.xml` rewrite 到 `/public/__sitemaps__/haloloan.xml`。这样彻底没有 cold start。

### 验证

```bash
for d in halofortune.com.au haloloan.com.au; do
  for i in 1 2 3; do
    cache=$(curl -sI --max-time 30 "https://$d/sitemap.xml" | grep -i "x-vercel-cache:" | tr -d '\r')
    t=$(curl -s -o /dev/null -w "%{time_total}" --max-time 30 "https://$d/sitemap.xml")
    echo "$d try $i: ${t}s   $cache"
  done
done
# 期望:
#   两域第 1 次都 < 5s
#   第 2-3 次都 < 0.5s 且 x-vercel-cache: HIT
```

---

## 2. 🟠 P0-2 残余 — ftfinance H1（及其它 data-lang-\* 元素）英文版对爬虫不可见

### 证据

```
$ curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/ \
    | python3 -c "import sys,re;h=sys.stdin.read();m=re.findall(r'<h1[^>]*>(.*?)</h1>',h,re.S);print(repr(m[0][:300]))"
'<span ... data-lang-zh="用心贷款，" data-lang-en="Lending with care,">用心贷款，</span>...'
```

H1 现在**渲染了中文默认文案**（修复前是完全空的，已经进步），所以验证脚本 V3 看起来通过。但仔细看 raw HTML：

- 英文文案 `"Lending with care,"` **只存在于 `data-lang-en` 属性里**
- 没有独立可见的 `<span lang="en">Lending with care,</span>` 元素

后果：
- 不执行 JS 的爬虫（GPTBot / ClaudeBot / CCBot / Common Crawl）抓到的 H1 **只有中文**
- 英文 H1 完全不进 AI 训练 + 不进 SEO snippet
- AI 在被英文 query 询问 Fingertips Finance 时，无法引用英文 H1 文本作为权威信源
- 这条问题大概率扩散到首页 H2/H3、CTA 按钮文本、产品介绍段落等任何当前用 `data-lang-zh`/`data-lang-en` 切换的元素上

### 修复

定位文件：`fingertips finance/scripts/inject-seo.mjs`（或对应的 SSR 注入模板/源 HTML）

**核心原则**：任何带可见文本的元素，**两种语言都要在初始 HTML 里以独立 DOM 节点存在**，CSS 控制显隐：

```html
<h1>
  <span lang="zh" class="lang-zh">用心贷款，一切尽在你掌中。</span>
  <span lang="en" class="lang-en">Lending with care, everything at your fingertips.</span>
</h1>
```

配套 CSS（应该全局加一次，所有页面共用）：
```css
/* 默认隐藏英文，html[lang="zh-AU"] 时显示中文 */
.lang-en { display: none; }
html[lang="en-AU"] .lang-en { display: inline; }
html[lang="en-AU"] .lang-zh { display: none; }

/* 对块级元素 */
h1 .lang-en, h2 .lang-en, h3 .lang-en, p .lang-en, div.bilingual .lang-en {
  display: none;
}
html[lang="en-AU"] :is(h1, h2, h3, p, div.bilingual) .lang-en {
  display: inline;
}
html[lang="en-AU"] :is(h1, h2, h3, p, div.bilingual) .lang-zh {
  display: none;
}
```

JS 切换语种时**只改 `<html lang="">` 属性**，CSS 自动接管显隐：
```js
function setLocale(loc) {
  document.documentElement.lang = loc === 'en' ? 'en-AU' : 'zh-AU';
  localStorage.setItem('locale', loc);
}
```

### 推广范围

应该一次性把首页所有以下元素都做成"双 span 并列"：
- 所有 H1 / H2 / H3
- 所有 CTA 按钮文本（"立即比价" / "Get Started"）
- Hero 段 / 卖点段 / 流程段的中英描述
- 产品卡片（home loan / construction loan / refinance / bridging / commercial）
- FAQ 问题与回答（如果 FAQ 模块也用 data-lang-\*）

**搜索整个静态站源代码里的 `data-lang-zh=` / `data-lang-en=`，每一处都重构**。

### 验证

```bash
echo "-- H1 应该含中英双语两段独立文本 --"
curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/ \
  | python3 -c "
import sys, re
h = sys.stdin.read()
m = re.findall(r'<h1[^>]*>(.*?)</h1>', h, re.S)
for x in m[:3]:
    spans = re.findall(r'<span[^>]*lang=\"([^\"]+)\"[^>]*>([^<]+)</span>', x)
    print('h1 langs/text:', spans)
"
# 期望: 看到 [('zh', '用心贷款...'), ('en', 'Lending with care...')]

echo ""
echo "-- 全站统计还剩多少 data-lang-* 元素 --"
n=$(curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/ | grep -oE 'data-lang-(zh|en)=' | wc -l | tr -d ' ')
echo "ftfinance home data-lang-* 元素数量: $n （期望: 0 或仅剩切换器 / 非可见文本）"

echo ""
echo "-- /products /how-it-works /quote 同样检查 --"
for p in products how-it-works quote lenders; do
  n=$(curl -sL -A "GPTBot/1.0" "https://ftfinance.com.au/$p" | grep -oE 'data-lang-(zh|en)=' | wc -l | tr -d ' ')
  echo "ftfinance/$p data-lang-* 数量: $n"
done
```

---

## 3. 🟡 P1-4 — halo\* HTML `<head>` 仍缺 hreflang 替代

### 证据

```
$ for d in halofortune.com.au haloloan.com.au; do
    echo "[$d]"
    curl -sL -A "GPTBot/1.0" "https://$d/" | grep -oE 'hreflang="[^"]+" href="[^"]+"'
  done
[halofortune.com.au]
(空)
[haloloan.com.au]
(空)
```

sitemap.xml 里有 `<xhtml:link rel="alternate" hreflang>`（前轮 Block G 看到过），但 HTML `<head>` 没补 —— 两侧信号不一致。Google 的 hreflang 信号优先级：HTML `<head>` > HTTP `Link:` header > sitemap。`<head>` 缺失等于浪费了 sitemap 的工作。

### 修复

定位文件：`src/lib/seo/metadata.ts` 的 `buildMetadata()` 函数

Next 14+ Metadata API 写法：
```ts
export function buildMetadata({ brand, locale, path }) {
  const base = brandUrl(brand);  // e.g. https://halofortune.com.au
  const cleanPath = path === '/' || path === '' ? '' : path;

  return {
    // ... 其它字段保持不变
    alternates: {
      canonical: `${base}/${locale}${cleanPath}`,
      languages: {
        'zh-AU':    `${base}/zh${cleanPath}`,
        'en-AU':    `${base}/en${cleanPath}`,
        'x-default': `${base}/zh${cleanPath}`,   // 默认面向中文社区
      },
    },
  };
}
```

### 验证

```bash
for d in halofortune.com.au haloloan.com.au; do
  echo "[$d /zh]"
  curl -sL -A "GPTBot/1.0" "https://$d/zh" | grep -oE 'hreflang="[^"]+" href="[^"]+"'
  echo "[$d /en]"
  curl -sL -A "GPTBot/1.0" "https://$d/en" | grep -oE 'hreflang="[^"]+" href="[^"]+"'
done
# 期望: 每个 URL 至少 3 行（zh-AU、en-AU、x-default）
# 且 zh-AU 指向 /zh、en-AU 指向 /en
```

也建议跑 schema validator 看 hreflang 配对是否对称：
- https://www.aleydasolis.com/english/international-seo-tools/hreflang-tags-generator/

---

## 4. 🟡 P1-9 — ftfinance `/blog` 索引缺 `Article` 与 `Organization` schema

### 证据

```
$ curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/blog | grep -oE '"@type":"[^"]+"' | sort -u
"@type":"BreadcrumbList"
"@type":"CollectionPage"
"@type":"ItemList"
"@type":"ListItem"
```

缺 `Article`、缺 `Organization`。对比 halo\* 两域 `/zh/blog` 都有 Article + Organization：
```
$ curl -sL -A "GPTBot/1.0" https://halofortune.com.au/zh/blog | grep -oE '"@type":"[^"]+"' | sort -u
"@type":"Article"           ← 有
"@type":"BreadcrumbList"
"@type":"CollectionPage"
"@type":"ItemList"
"@type":"ListItem"
"@type":"Organization"      ← 有
```

### 修复

定位：ftfinance `/blog` 是通过 `fingertips finance/vercel.json` reverse-proxy 到 v2-portal 的 brand-aware blog index 路由（brand=ftfinance）。需要改 v2-portal 端的 blog index template。

具体改两处：

**A. 每个 ItemList item 嵌入 Article schema**

把现有的：
```json
{
  "@type": "ListItem",
  "position": 1,
  "url": "https://ftfinance.com.au/blog/some-article"
}
```

升级成：
```json
{
  "@type": "ListItem",
  "position": 1,
  "url": "https://ftfinance.com.au/blog/some-article",
  "item": {
    "@type": "Article",
    "@id": "https://ftfinance.com.au/blog/some-article#article",
    "headline": "<文章标题>",
    "url": "https://ftfinance.com.au/blog/some-article",
    "datePublished": "2026-04-15T...",
    "dateModified": "2026-05-01T...",
    "image": "https://ftfinance.com.au/blog/some-article/cover.jpg",
    "author": {
      "@type": "Organization",
      "@id": "https://ftfinance.com.au/#organization"
    },
    "publisher": {
      "@type": "Organization",
      "@id": "https://ftfinance.com.au/#organization"
    },
    "inLanguage": "zh-AU"
  }
}
```

注意：**作者必须是 Organization 引用 ftfinance 自己的 `#organization` @id，不能引用 Wei Ming Chen 或任何 halo\* 实体（防火墙）**。如果 ftfinance 有自己的 principal broker 想暴露姓名，可以另起一个 ftfinance 域名下的 Person @id（如 `https://ftfinance.com.au/#principal-broker`），但**绝对不要**跨域引用 halo\* 的 Person。

**B. 页面级加 Organization publisher**

在 `CollectionPage` 节点里加：
```json
{
  "@type": "CollectionPage",
  "@id": "https://ftfinance.com.au/blog#page",
  "url": "https://ftfinance.com.au/blog",
  "isPartOf": { "@id": "https://ftfinance.com.au/#website" },
  "publisher": { "@id": "https://ftfinance.com.au/#organization" },
  "mainEntity": { "@id": "https://ftfinance.com.au/blog#itemlist" }
}
```

并在同一 `@graph` 数组里包含 Organization 节点（与 ftfinance 首页 #organization 同 @id，可以 minimal 引用）：
```json
{
  "@type": "Organization",
  "@id": "https://ftfinance.com.au/#organization",
  "name": "Fingertips Finance"
}
```
完整 Organization 节点已在首页声明，这里短引用即可（Google 会跨页面合并同 @id 的实体）。

### 验证

```bash
curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/blog | grep -oE '"@type":"[^"]+"' | sort -u
# 期望: 出现 "Article" 与 "Organization"

echo "-- Article 节点的 author/publisher 应引用 ftfinance #organization --"
curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/blog \
  | python3 -c "
import sys, re, json
html = sys.stdin.read()
for m in re.finditer(r'<script type=\"application/ld\+json\">(.+?)</script>', html, re.S):
    try:
        data = json.loads(m.group(1))
        # 递归找 Article
        def walk(o):
            if isinstance(o, dict):
                if o.get('@type') == 'Article':
                    print('Article author:', o.get('author'))
                    print('Article publisher:', o.get('publisher'))
                for v in o.values(): walk(v)
            elif isinstance(o, list):
                for v in o: walk(v)
        walk(data)
    except: pass
"
# 期望: author/publisher 都引用 {@id: 'https://ftfinance.com.au/#organization'}
# 绝对不能出现 halofortune.com.au 或 wei-ming-chen 等关联
```

---

## 5. 防火墙抽查（实施后必跑）

```bash
echo "=== 改 ftfinance /blog Article 之后再跑一次防火墙 ==="
curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/blog \
  | grep -ciE "halofortune|haloloan|Halo Fortune|Halo Loan|wei-ming"
# 期望: 0

echo ""
echo "=== 改 halo* hreflang 之后确认它们仍不提及 ftfinance ==="
for d in halofortune.com.au haloloan.com.au; do
  curl -sL -A "GPTBot/1.0" "https://$d/" \
    | grep -ciE "ftfinance|fingertips|掌忠贷"
done
# 期望: 两域均 0
```

---

## 6. 完整验证脚本（在前文 §7 V1-V12 基础上加新检查）

跑前置文档 `SEO-GEO-OPTIMIZATION-2026-05-11.md` §7 全部 V1-V12 + 下面 F1-F4：

```bash
echo "########## F1: sitemap 缓存命中（haloloan 不再 timeout） ##########"
for d in halofortune.com.au haloloan.com.au; do
  for i in 1 2 3; do
    out=$(curl -s -o /dev/null -w "%{time_total} %{http_code}" --max-time 30 "https://$d/sitemap.xml")
    cache=$(curl -sI --max-time 30 "https://$d/sitemap.xml" | grep -i "x-vercel-cache:" | tr -d '\r' | xargs)
    echo "$d try $i: $out   $cache"
  done
done
# 期望: 两域所有尝试均 < 5s 且 status=200

echo ""
echo "########## F2: ftfinance H1 双语并列 ##########"
curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/ \
  | python3 -c "
import sys, re
h = sys.stdin.read()
m = re.findall(r'<h1[^>]*>(.*?)</h1>', h, re.S)
for i, x in enumerate(m[:2]):
    spans = re.findall(r'<span[^>]*lang=\"([^\"]+)\"[^>]*>([^<]+)</span>', x)
    print(f'h1[{i}]:', spans)
"
# 期望: 至少 [('zh', '...'), ('en', '...')] 这样的双语并列

echo ""
echo "########## F3: halo* head 含 hreflang ##########"
for d in halofortune.com.au haloloan.com.au; do
  echo "[$d /zh]"
  curl -sL -A "GPTBot/1.0" "https://$d/zh" | grep -oE 'hreflang="[^"]+" href="[^"]+"'
done
# 期望: 每域 3 行（zh-AU、en-AU、x-default）

echo ""
echo "########## F4: ftfinance /blog Article + Organization ##########"
curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/blog | grep -oE '"@type":"[^"]+"' | sort -u
# 期望: 含 "Article" 与 "Organization"
```

---

## 7. 实施顺序建议

| 优先级 | 任务 | 预估工作量 |
|---|---|---|
| 立即 | §1 — haloloan sitemap fix（去掉 force-static + 加 Cache-Control 头） | 1 小时 |
| 本周 | §2 — ftfinance H1 与所有 `data-lang-*` 元素改双 span 并列（含全站推广） | 半天到一天 |
| 2 周内 | §3 — halo\* metadata 补 hreflang | 30 分钟 |
| 2 周内 | §4 — ftfinance `/blog` 加 Article + Organization schema | 1-2 小时 |

完成后跑前置文档 §7 全部 V1-V12 + 本文件 §6 F1-F4，每条"期望"通过即收工。

---

## 8. 不要做的事（提醒）

- ❌ 不要为了"解决 ftfinance /blog 缺 Article 问题"在 author 字段引用 Wei Ming Chen 或 halo\* 实体（防火墙）
- ❌ 不要把 ftfinance 静态站的 H1 重构成 "中文默认 + JS 注入英文" —— 必须**两段独立 DOM 节点同时存在**
- ❌ 不要在 sitemap.ts 上保留 `dynamic = 'force-static'`（haloloan 回归的元凶）
- ❌ 不要把 halo\* 的 hreflang 写成同一 URL（参考 ftfinance 之前的错误模式）

---

**文档结束。完成这 4 条 + 1 条 P0 回归修复后，三域的 SEO/GEO 工程化质量将达到完全顶尖水平（18/18）。**
