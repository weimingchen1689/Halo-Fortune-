# SEO/GEO 优化 — Round 3 Followup（2026-05-11 18:25）

**前置文档**：
- Round 1: [`SEO-GEO-OPTIMIZATION-2026-05-11.md`](./SEO-GEO-OPTIMIZATION-2026-05-11.md)
- Round 2: [`SEO-GEO-OPTIMIZATION-2026-05-11-FOLLOWUP.md`](./SEO-GEO-OPTIMIZATION-2026-05-11-FOLLOWUP.md)

**Round 3 完成度**：16 / 18 明确项 + 2 部分 + 1 新发现 = 实质 88%

本次 Round 3 verify 后修复完成的项目（继 Round 2 后新增）：
- ✅ **haloloan sitemap 回归** —— 从 15s timeout 修复到 0.13s + cache HIT，URL count 172
- ✅ **ftfinance 首页 H1** —— 双 span 双语 SSR 已完美实现（class="lang-zh" + class="lang-en"）
- ✅ **ftfinance /blog Organization schema** —— 已添加

剩下 3 条没做透 + 1 条新发现，本文件覆盖。预估总工作量 < 4 小时。

合规防火墙规则不变（halo\* ↔ ftfinance 不能在公开层互相提及）—— 见 Round 1 文档 §0、§1、§8。

---

## 1. 🟠 P0-2 expand — ftfinance 全站 ~984 个 `data-lang-*` 元素仍待迁移

### 证据

```
$ for p in "" products how-it-works lenders quote about contact; do
    n=$(curl -sL -A "GPTBot/1.0" "https://ftfinance.com.au/$p" | grep -oE 'data-lang-(zh|en)=' | wc -l)
    echo "/$p data-lang-* count: $n"
  done

/             count: 252
/products     count: 292
/how-it-works count: 84
/lenders      count: 28
/quote        count: 72
/about        count: 116
/contact      count: 140
              ─────
              合计 ~984 个
```

首页 H1 已经做得很完美（class="lang-zh" / class="lang-en" 独立 DOM 节点，双语都在 SSR HTML 里），但**其它 H2/H3/CTA/产品段/FAQ/流程说明等 ~984 个元素仍在用旧模式** —— `data-lang-zh` / `data-lang-en` 当 attribute 存数据，JS 动态注入到 textContent。

### 后果

- 不执行 JS 的爬虫（GPTBot / ClaudeBot / CCBot / Common Crawl / Bytespider）抓到的页面**只有默认语种文本**，另一语种**完全不可见**
- `/products` 292 个 data-lang 意味着产品介绍英文版基本没进 AI 训练
- 单页 SSR payload 浪费一半（数据存在但抽取不到）

### 修复

**核心原则**：把所有 `data-lang-zh="X" data-lang-en="Y">默认</tag>` 模式替换为：

```html
<tag>
  <span lang="zh-AU" class="lang-zh">X</span>
  <span lang="en-AU" class="lang-en">Y</span>
</tag>
```

CSS 全局规则（应该已经在首页 H1 修复时加好，确认一下）：
```css
.lang-en { display: none; }
html[lang="en-AU"] .lang-en { display: inline; }
html[lang="en-AU"] .lang-zh { display: none; }

/* 块级元素的情况 */
:is(h1, h2, h3, p, div) > .lang-en { display: none; }
html[lang="en-AU"] :is(h1, h2, h3, p, div) > .lang-en { display: inline; }
html[lang="en-AU"] :is(h1, h2, h3, p, div) > .lang-zh { display: none; }
```

JS 切换语种时**只改 `<html lang="">`**，CSS 自动接管显隐：
```js
function setLocale(loc) {
  document.documentElement.lang = loc === 'en' ? 'en-AU' : 'zh-AU';
  localStorage.setItem('locale', loc);
}
```

### 实施步骤建议

1. **批量重构脚本** —— 在 `fingertips finance/scripts/inject-seo.mjs` 或源静态 HTML 上跑一次 codemod（regex 替换 + AST 分析），把所有 `<XXX data-lang-zh="A" data-lang-en="B">A</XXX>` 转成 `<XXX><span lang="zh-AU" class="lang-zh">A</span><span lang="en-AU" class="lang-en">B</span></XXX>`
2. 删除依赖 `data-lang-*` 切换的旧 JS（保留 `setLocale()` 函数即可）
3. 跑验证脚本（见下面）

### 验证

```bash
echo "########## 全站 data-lang-* 元素清零验证 ##########"
for p in "" products how-it-works lenders quote about contact; do
  n=$(curl -sL -A "GPTBot/1.0" --max-time 15 "https://ftfinance.com.au/$p" \
       | grep -oE 'data-lang-(zh|en)=' | wc -l | tr -d ' ')
  echo "ftfinance.com.au/$p data-lang-* count: $n"
done
# 期望: 全部 0（或仅剩用作语种切换 toggle 的少量 attribute, ≤ 5 个）

echo ""
echo "########## 双 span 抽样验证（/products） ##########"
curl -sL -A "GPTBot/1.0" --max-time 15 https://ftfinance.com.au/products \
  | python3 -c "
import sys, re
h = sys.stdin.read()
zh = len(re.findall(r'<span[^>]*class=\"lang-zh\"', h))
en = len(re.findall(r'<span[^>]*class=\"lang-en\"', h))
print(f'lang-zh spans: {zh}')
print(f'lang-en spans: {en}')
print(f'match (zh == en): {zh == en}')
"
# 期望: zh 和 en 数量相等，且 > 50（说明产品页所有可见文本都做了双 span）
```

---

## 2. 🟡 P1-4 — halo\* HTML `<head>` 仍完全没有 `hreflang` 替代

### 证据（**毫无进展**，与 Round 2 一致）

```
$ for u in https://halofortune.com.au/zh https://halofortune.com.au/en https://haloloan.com.au/zh https://haloloan.com.au/en; do
    echo "[$u]"
    curl -sL -A "GPTBot/1.0" "$u" | grep -oE 'hreflang="[^"]+" href="[^"]+"'
  done
[https://halofortune.com.au/zh]
(空)
[https://halofortune.com.au/en]
(空)
[https://haloloan.com.au/zh]
(空)
[https://haloloan.com.au/en]
(空)
```

sitemap.xml 有 `<xhtml:link rel="alternate" hreflang>`，但 HTML `<head>` 一直没补。已经在 Round 2 followup §3 说明，**两轮都没动这块**。

### 修复（精确到代码）

定位文件：`src/lib/seo/metadata.ts`

Round 2 followup §3 的代码模板照搬。检查 `buildMetadata()` 函数当前实现，找到返回的 `Metadata` 对象（Next.js 14+ App Router 标准接口），把 `alternates` 字段补全：

```ts
import type { Metadata } from 'next';

interface BuildMetadataArgs {
  brand: BrandId;     // 'halofortune' | 'haloloan' | 'ftfinance'
  locale: 'zh' | 'en';
  path: string;       // e.g. '/blog' or '' for homepage
}

export function buildMetadata({ brand, locale, path }: BuildMetadataArgs): Metadata {
  const base = brandUrl(brand);  // 假设已有，返回 https://halofortune.com.au 等
  const cleanPath = path === '/' || path === '' ? '' : path;

  return {
    // ... title, description, openGraph 等保持不变

    alternates: {
      canonical: `${base}/${locale}${cleanPath}`,
      languages: {
        'zh-AU':     `${base}/zh${cleanPath}`,
        'en-AU':     `${base}/en${cleanPath}`,
        'x-default': `${base}/zh${cleanPath}`,   // 默认面向中文社区
      },
    },
  };
}
```

**关键点**：
- ftfinance 已经是 `x-default only`（Round 1 修复后），**不要碰 ftfinance 的 metadata**
- 只改 halo\* brand 的逻辑分支
- 如果当前 `buildMetadata` 是 brand-aware 的，确保 ftfinance 走自己的（只输出 x-default）路径，halo\* 走新加的（三个 hreflang）路径

### 验证

```bash
echo "########## halo* head 含 hreflang ##########"
for u in https://halofortune.com.au/zh https://halofortune.com.au/en https://haloloan.com.au/zh https://haloloan.com.au/en; do
  echo "[$u]"
  curl -sL -A "GPTBot/1.0" --max-time 15 "$u" | grep -oE 'hreflang="[^"]+" href="[^"]+"'
done
# 期望: 每个 URL 输出 3 行：
#   hreflang="zh-AU" href=".../zh{cleanPath}"
#   hreflang="en-AU" href=".../en{cleanPath}"
#   hreflang="x-default" href=".../zh{cleanPath}"

echo ""
echo "########## ftfinance hreflang 应保持仅 x-default ##########"
curl -sL -A "GPTBot/1.0" --max-time 15 https://ftfinance.com.au/ \
  | grep -oE 'hreflang="[^"]+" href="[^"]+"'
# 期望: 仅 1 行（x-default）
```

也建议跑 hreflang 配对一致性 validator：
- https://www.aleydasolis.com/english/international-seo-tools/hreflang-tags-generator/
- https://technicalseo.com/tools/hreflang/

---

## 3. 🟡 P1-9 part B — ftfinance `/blog` 索引仍缺 `Article` schema（Organization 已补）

### 证据

```
$ curl -sL -A "GPTBot/1.0" https://ftfinance.com.au/blog | grep -oE '"@type":"[^"]+"' | sort -u
"@type":"BreadcrumbList"
"@type":"CollectionPage"
"@type":"ContactPoint"
"@type":"Country"
"@type":"ItemList"
"@type":"ListItem"
"@type":"Organization"     ← ✅ 这轮加上
"@type":"PostalAddress"
"@type":"PropertyValue"
                            ↑ 仍然没有 "Article"
```

对比 halo\* 两域 `/zh/blog` 都有 Article + Organization。

防火墙抽查通过：`TOTAL Article nodes: 0, halo* leaks: 0` —— 因为 Article 还没渲染所以无机会泄漏。这反而说明实现没动。

### 修复

定位：ftfinance `/blog` 由 `fingertips finance/vercel.json` reverse-proxy 到 v2-portal 的 brand-aware blog index 路由（`brand=ftfinance`）。

在 v2-portal 端的 blog index template 里，给每个 `ItemList.itemListElement` 嵌入 Article 节点：

```json
{
  "@type": "ListItem",
  "position": 1,
  "url": "https://ftfinance.com.au/blog/<slug>",
  "item": {
    "@type": "Article",
    "@id": "https://ftfinance.com.au/blog/<slug>#article",
    "headline": "<文章标题>",
    "url": "https://ftfinance.com.au/blog/<slug>",
    "datePublished": "<ISO 8601>",
    "dateModified": "<ISO 8601>",
    "image": "https://ftfinance.com.au/blog/<slug>/cover.jpg",
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

### ⛔ 防火墙绝对禁令（再强调）

- `author` 和 `publisher` **必须**引用 `https://ftfinance.com.au/#organization`
- **绝对不能**引用：
  - `https://halofortune.com.au/#organization`
  - `https://haloloan.com.au/#organization`
  - `https://halofortune.com.au/#wei-ming-chen` 或任何 halo\* 的 Person @id
  - 字符串 "Halo Fortune" / "Halo Loan" / "Wei Ming Chen"
- 如果 ftfinance 想暴露具体人物作者，需要新建 ftfinance 域名下的 Person @id（如 `https://ftfinance.com.au/#principal-broker`），**独立**于 halo\* 的 Person 节点

### 验证

```bash
echo "########## ftfinance /blog Article schema ##########"
curl -sL -A "GPTBot/1.0" --max-time 15 https://ftfinance.com.au/blog \
  | grep -oE '"@type":"[^"]+"' | sort -u
# 期望: 输出含 "Article"

echo ""
echo "########## ftfinance /blog Article author/publisher 防火墙 ##########"
curl -sL -A "GPTBot/1.0" --max-time 15 https://ftfinance.com.au/blog \
  | python3 -c "
import sys, re, json
html = sys.stdin.read()
arts, leaks = 0, 0
for m in re.finditer(r'<script type=\"application/ld\+json\">(.+?)</script>', html, re.S):
    try:
        data = json.loads(m.group(1))
        def walk(o):
            global arts, leaks
            if isinstance(o, dict):
                if o.get('@type') == 'Article':
                    arts += 1
                    a, p = o.get('author'), o.get('publisher')
                    blob = json.dumps([a, p]).lower()
                    if any(s in blob for s in ['halofortune','haloloan','wei-ming','halo fortune','halo loan']):
                        leaks += 1
                        print(f'  LEAK detected: author={a} publisher={p}')
                for v in o.values(): walk(v)
            elif isinstance(o, list):
                for v in o: walk(v)
        walk(data)
    except: pass
print(f'TOTAL Article nodes: {arts}, halo* leaks: {leaks}')
"
# 期望: Article nodes > 0, halo* leaks: 0
```

---

## 4. 🟡 NEW — ftfinance sitemap 完全没有 blog 文章页 URL

### 证据

```
$ curl -s https://ftfinance.com.au/sitemap.xml | grep -oE '<loc>[^<]+</loc>'
<loc>https://ftfinance.com.au/</loc>
<loc>https://ftfinance.com.au/products</loc>
<loc>https://ftfinance.com.au/how-it-works</loc>
<loc>https://ftfinance.com.au/lenders</loc>
<loc>https://ftfinance.com.au/quote</loc>
<loc>https://ftfinance.com.au/blog</loc>           ← 索引页有
<loc>https://ftfinance.com.au/about</loc>
<loc>https://ftfinance.com.au/contact</loc>
<loc>https://ftfinance.com.au/legal/credit-guide</loc>
<loc>https://ftfinance.com.au/legal/privacy</loc>
<loc>https://ftfinance.com.au/legal/terms</loc>
<loc>https://ftfinance.com.au/legal/complaints</loc>

URL count: 12 （但 0 篇博客文章页）
```

对比：halofortune sitemap 68 URL（含每篇博客的 zh + en 共两份），haloloan sitemap **172 URL**（最丰富）。ftfinance 仅 12 个 section URL，**0 篇博客文章**。

### 后果

- Google / Bing 通过 sitemap **发现不到**任何 ftfinance 博客文章
- 只能通过 /blog 索引页爬出来 —— 即使 §3 修好了 Article schema，发现效率仍然比 sitemap 直接列出差
- 浪费了 v2-portal 现有内容的 SEO 价值

### 修复

ftfinance 的 sitemap 由 `fingertips finance/` 静态站的某个脚本生成（推测 `scripts/generate-sitemap.mjs` 或类似）。需要从 v2-portal API 拉取 `brand=ftfinance` 的文章列表，注入到 sitemap.xml。

```ts
// fingertips finance/scripts/generate-sitemap.mjs
import { fetchFtFinanceArticles } from './v2-portal-client.mjs';

const sectionUrls = [
  { loc: 'https://ftfinance.com.au/', changefreq: 'weekly', priority: 1.0 },
  { loc: 'https://ftfinance.com.au/products', changefreq: 'monthly', priority: 0.9 },
  // ... 现有 12 个
];

const articles = await fetchFtFinanceArticles();   // 从 v2-portal API
const articleUrls = articles.map(a => ({
  loc: `https://ftfinance.com.au/blog/${a.slug}`,
  lastmod: a.updatedAt,
  changefreq: 'monthly',
  priority: 0.7,
}));

const allUrls = [...sectionUrls, ...articleUrls];
// 渲染成 sitemap.xml
```

或者用 sitemap index 模式，把 v2-portal 的 `brand=ftfinance` 子 sitemap 引用进来：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>https://ftfinance.com.au/sitemap-sections.xml</loc>
  </sitemap>
  <sitemap>
    <loc>https://ftfinance.com.au/sitemap-blog.xml</loc>
  </sitemap>
</sitemapindex>
```
然后 `/sitemap-blog.xml` 路径通过 vercel.json reverse-proxy 到 v2-portal 的 `?brand=ftfinance&type=blog-sitemap`。

### 验证

```bash
echo "########## ftfinance sitemap 包含博客文章页 ##########"
cnt=$(curl -s --max-time 15 https://ftfinance.com.au/sitemap.xml \
       | grep -oE 'https://ftfinance\.com\.au/blog/[a-z0-9-]+' | wc -l | tr -d ' ')
echo "ftfinance sitemap 中的 blog post URL 数量: $cnt"
# 期望: > 0（应等于 v2-portal 中 brand=ftfinance 的发布文章数）

echo ""
echo "########## sample blog URL ##########"
curl -s --max-time 15 https://ftfinance.com.au/sitemap.xml \
  | grep -oE 'https://ftfinance\.com\.au/blog/[a-z0-9-]+' | head -5
```

---

## 5. 完整验证脚本（接 Round 1 V1-V12 + Round 2 F1-F4 之后）

跑前两轮的所有 V/F 验证后，再补这一轮的 G1-G4：

```bash
echo "########## G1: ftfinance 全站 data-lang-* 清零 ##########"
for p in "" products how-it-works lenders quote about contact; do
  n=$(curl -sL -A "GPTBot/1.0" --max-time 15 "https://ftfinance.com.au/$p" \
       | grep -oE 'data-lang-(zh|en)=' | wc -l | tr -d ' ')
  echo "ftfinance.com.au/$p data-lang-* count: $n  (expected 0 or ≤5)"
done

echo ""
echo "########## G2: halo* head 含 3 个 hreflang ##########"
for u in https://halofortune.com.au/zh https://halofortune.com.au/en https://haloloan.com.au/zh https://haloloan.com.au/en; do
  cnt=$(curl -sL -A "GPTBot/1.0" --max-time 15 "$u" | grep -cE 'hreflang="[^"]+" href="[^"]+"')
  echo "$u  hreflang lines: $cnt  (expected 3)"
done

echo ""
echo "########## G3: ftfinance /blog 含 Article 且无防火墙泄漏 ##########"
curl -sL -A "GPTBot/1.0" --max-time 15 https://ftfinance.com.au/blog \
  | python3 -c "
import sys, re, json
html = sys.stdin.read()
types = sorted(set(re.findall(r'\"@type\":\"([^\"]+)\"', html)))
has_article = 'Article' in types
arts, leaks = 0, 0
for m in re.finditer(r'<script type=\"application/ld\+json\">(.+?)</script>', html, re.S):
    try:
        data = json.loads(m.group(1))
        def walk(o):
            global arts, leaks
            if isinstance(o, dict):
                if o.get('@type') == 'Article':
                    arts += 1
                    blob = json.dumps([o.get('author'), o.get('publisher')]).lower()
                    if any(s in blob for s in ['halofortune','haloloan','wei-ming','halo fortune','halo loan']):
                        leaks += 1
                for v in o.values(): walk(v)
            elif isinstance(o, list):
                for v in o: walk(v)
        walk(data)
    except: pass
print(f'Article in @types: {has_article}')
print(f'Article nodes: {arts}  (expected > 0)')
print(f'halo* leaks: {leaks}  (expected 0)')
"

echo ""
echo "########## G4: ftfinance sitemap 含 blog post URLs ##########"
cnt=$(curl -s --max-time 15 https://ftfinance.com.au/sitemap.xml \
       | grep -oE 'https://ftfinance\.com\.au/blog/[a-z0-9-]+' | wc -l | tr -d ' ')
echo "ftfinance sitemap blog post URL count: $cnt  (expected > 0)"

echo ""
echo "########## G5: 防火墙最终抽查 ##########"
for d in halofortune.com.au haloloan.com.au; do
  for p in "" zh en zh/blog en/blog; do
    n=$(curl -sL -A "GPTBot/1.0" --max-time 15 "https://$d/$p" 2>/dev/null \
         | grep -ciE "ftfinance|fingertips|掌忠贷")
    echo "$d/$p mentions ftfinance: $n (expected 0)"
  done
done
for p in "" blog products quote about; do
  n=$(curl -sL -A "GPTBot/1.0" --max-time 15 "https://ftfinance.com.au/$p" 2>/dev/null \
       | grep -ciE "halofortune|haloloan|Halo Fortune|Halo Loan|wei-ming")
  echo "ftfinance.com.au/$p mentions halo*: $n (expected 0)"
done
```

---

## 6. 不要做的事（再次强调）

1. ❌ **不要**在 ftfinance /blog 的 Article schema 里把 author/publisher 引用到 halo\* 实体或 Wei Ming Chen
2. ❌ **不要**在数据迁移时把 ftfinance 的双 span 改成"中文默认 + JS 注入英文"（必须**两段独立 DOM 节点同时存在**）
3. ❌ **不要**碰 ftfinance 的 hreflang（应保持只有 x-default 一行，Round 1 已修对）
4. ❌ **不要**为了 sitemap 加 blog URL 而把 halo\* 的博客文章 URL 误加到 ftfinance sitemap

---

## 7. 实施顺序建议

| 优先级 | 任务 | 工作量 | 文件 |
|---|---|---|---|
| 🟠 | §1 — ftfinance 全站 data-lang 双 span 迁移 | 半天（最大块） | `fingertips finance/` 静态 HTML + inject-seo.mjs |
| 🟡 | §2 — halo\* metadata 补 hreflang | 30 分钟 | `src/lib/seo/metadata.ts` |
| 🟡 | §3 — ftfinance /blog Article schema | 1 小时 | v2-portal blog index template (`brand=ftfinance` 分支) |
| 🟡 | §4 — ftfinance sitemap 注入 blog post URLs | 1 小时 | `fingertips finance/scripts/generate-sitemap.*` |

完成后跑 §5 G1-G5 + 前两轮的 V1-V12 + F1-F4，所有"期望"通过即收工。

---

**文档结束。完成这 4 条后，三域 SEO/GEO 工程化质量达到 18/18（明确项）+ 1（新发现）= 100% 顶尖水平。**
