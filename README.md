# Halo-Fortune-

SEO & AI-crawler audit and ready-to-deploy templates for
[halofortune.com.au](https://halofortune.com.au/).

## Contents

```
docs/
  SEO-AUDIT.md            # 审计报告 + 修复优先级清单 + 自查命令
seo/
  robots.txt              # 显式放行 GPTBot / ClaudeBot / PerplexityBot ...
  llms.txt                # AI 友好的站点摘要
  sitemap.xml             # URL 模板（含 hreflang）
  head-snippet.html       # <head> 完整模板（title / OG / Twitter / canonical）
  structured-data/
    organization.jsonld   # 全站 Organization
    website.jsonld        # WebSite + SearchAction
    service.jsonld        # 服务页示例
    breadcrumb.jsonld     # 面包屑示例
    faqpage.jsonld        # FAQ 示例
```

## 部署步骤

1. **静态文件**：把 `seo/robots.txt`、`seo/llms.txt`、`seo/sitemap.xml`
   原样放到站点根目录，可通过下面路径访问：
   - `https://halofortune.com.au/robots.txt`
   - `https://halofortune.com.au/llms.txt`
   - `https://halofortune.com.au/sitemap.xml`
2. **HTML head**：把 `seo/head-snippet.html` 内容合并进每页 `<head>`，
   替换 `{{ page_title }}` / `{{ page_description }}` / `{{ path }}` 等占位符。
3. **JSON-LD**：把 `seo/structured-data/organization.jsonld` 与
   `website.jsonld` 注入每页 `<head>`；按页型追加 `service.jsonld` /
   `breadcrumb.jsonld` / `faqpage.jsonld`。
4. **GSC / Bing**：在 Google Search Console、Bing Webmaster Tools
   提交 sitemap，验证站点所有权。

完整的修复优先级、复验命令与发现详情见 [`docs/SEO-AUDIT.md`](docs/SEO-AUDIT.md)。
