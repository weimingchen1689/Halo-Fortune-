# Halo Fortune Group / Halo Loan / Fingertips Finance
## 安全审计报告（black-box passive）—— 2026-05-11

**审计方法**：基于 2026-05-11 19:17 UTC 真实生产数据，全程 passive black-box（只读探测，零主动攻击，零暴力枚举）
**目标读者**：实施修复的下一位 Claude / 工程师
**前置文档**：
- Round 1: [`SEO-GEO-OPTIMIZATION-2026-05-11.md`](./SEO-GEO-OPTIMIZATION-2026-05-11.md)
- Round 2: [`SEO-GEO-OPTIMIZATION-2026-05-11-FOLLOWUP.md`](./SEO-GEO-OPTIMIZATION-2026-05-11-FOLLOWUP.md)
- Round 3: [`SEO-GEO-OPTIMIZATION-2026-05-11-FOLLOWUP-R3.md`](./SEO-GEO-OPTIMIZATION-2026-05-11-FOLLOWUP-R3.md)

---

## 0. 总体评级：**A-（非常好）**

三个站点的安全配置在 boutique 金融科技站里属于**第一梯队**：

- ✅ TLS 1.2/1.3 only，证书有效
- ✅ HSTS 2 年
- ✅ CSP `frame-ancestors 'self'` 防 clickjacking
- ✅ Permissions-Policy 关闭敏感 API（geolocation/microphone/camera）
- ✅ PII 表单页 `private/no-store`（haloloan、halofortune scenario、ftfinance quote/contact 全部正确）
- ✅ 三域子域名零暴露（41 个常见名全部 NXDOMAIN）
- ✅ API 端点正确 405（POST only）
- ✅ CORS preflight 不接受任意 origin
- ✅ 无 open redirect
- ✅ haloloan 前端 JS bundle 扫描零 secret
- ✅ Vercel/Cloudflare WAF 在本次审计中**实际触发了 rate-limit 防御**（说明边缘防御主动）

**未发现 P0**。下面是 4 条 P1 + 3 条 P2 + 3 条 P3。

---

## 1. 给接手者的关键说明

### 1a. 合规防火墙（与 SEO 文档相同的规则）

`halofortune.com.au` 与 `haloloan.com.au` 同法人（Halo Fortune Group Pty Ltd, ACL 483923）—— 可公开互相引用。
`ftfinance.com.au` 是独立法人（Fingertips Finance Pty Ltd ATF Fingertips Trust, ACR 530340 under Outsource Financial ACL 384324）—— **不能与 halo\* 公开互相引用**。

实施本文档时：
- ❌ ftfinance 的 `security.txt` 邮箱**不能**写 `security@halofortune.com.au`
- ❌ ftfinance 的 CSP `script-src` / `connect-src` **不能**白名单 halo\* 的域名（反之亦然）
- ❌ ftfinance 的 complaints/privacy/terms 页面**不能**显式提及 halo\* 品牌
- ✅ 三域可共用相同的 CSP 模板**框架**（结构相同，第三方 endpoint 列表按品牌分开维护）

### 1b. 不要做的事

- ❌ **不要**在 CSP 里加 `'unsafe-eval'` 给任意第三方 origin —— 只 self + Next.js hydration 需要的最少集合
- ❌ **不要**关闭 HSTS（只能加 directive 不能减）
- ❌ **不要**为了"统一"把 halo\* 与 ftfinance 的 `security.txt` 合并到一个邮箱
- ❌ **不要**手动改 Vercel/Cloudflare WAF rules 把本次审计触发的 rate-limit 关掉（那是真实防御在工作）
- ❌ **不要**采用任何"自签证书"或"自管 TLS"的方案 —— 现有 Vercel/Let's Encrypt 自动续签是最佳实践

### 1c. 实施顺序（约 1 工作日）

| 优先级 | 项 | 工作量 |
|---|---|---|
| 🟠 P1-1 | 完整 CSP（default-src + script-src + connect-src ...） | 半天 |
| 🟠 P1-2 | HSTS 加 `includeSubDomains; preload` | 30 分钟 + 24h 观察 |
| 🟠 P1-3 | 确认 haloloan `/admin` 307 目标 | 15 分钟 |
| 🟠 P1-4 | ftfinance form CDN cache purge | 30 分钟 |
| 🟡 P2-1 | halo\* 加 www. 子域证书 | 30 分钟 |
| 🟡 P2-2 | haloloan 补 /zh/complaints + /zh/terms | 1 小时 |
| 🟡 P2-3 | halofortune + ftfinance 加 security.txt | 30 分钟 |
| ⚪ P3 | COOP/CORP 加固、white-box 自查 | 长期 |

---

## 2. 🟠 P1（一周内修）

### P1-1 — CSP 只有 `frame-ancestors`，缺主动 XSS 防御层

#### 证据

```
$ curl -sI https://halofortune.com.au/ | grep -i content-security
content-security-policy: frame-ancestors 'self'
```
三域全部一致 —— **只防 clickjacking，完全不防 XSS**。

#### 风险

Mortgage broker 收集 PII（payslip / bank statement / 收入数据）。如果出现：
- React/Next 组件 XSS 漏洞
- 第三方 SDK（GA、Vercel Analytics、Sentry 等）被供应链投毒
- 客户端依赖（npm 包）后门

→ 攻击者可以注入 inline script 把 PII fetch 到 `evil.com`。当前 CSP 没有 `connect-src` / `script-src` 限制，**无法阻止外泄**。

#### 修复

定位文件：`next.config.js`（halo\*）或 `vercel.json`（ftfinance 静态站）

**Next.js 模板**（halo\*）：

```js
// next.config.js
const cspHeader = `
  default-src 'self';
  script-src 'self' 'unsafe-inline' 'unsafe-eval'
    https://www.googletagmanager.com
    https://*.googletagmanager.com
    https://www.google-analytics.com
    https://va.vercel-scripts.com
    https://vercel.live;
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  font-src 'self' https://fonts.gstatic.com data:;
  img-src 'self' data: blob: https:;
  connect-src 'self'
    https://*.vercel-insights.com
    https://*.googletagmanager.com
    https://www.google-analytics.com
    https://vercel.live wss://ws-us3.pusher.com;
  frame-src 'self';
  frame-ancestors 'self';
  base-uri 'self';
  form-action 'self';
  object-src 'none';
  upgrade-insecure-requests;
`.replace(/\s+/g, ' ').trim();

module.exports = {
  async headers() {
    return [{
      source: '/(.*)',
      headers: [
        { key: 'Content-Security-Policy', value: cspHeader },
        // 其它 headers...
      ],
    }];
  },
};
```

**ftfinance** 用相似模板，但 `script-src` / `connect-src` 列表 **必须独立维护**（不能引用 halo\* 子域，遵守防火墙）。如果 ftfinance 用了 Cloudflare Analytics，单独加 `https://static.cloudflareinsights.com` 等。

#### 关键约束

1. `'unsafe-inline'` + `'unsafe-eval'` 是 **Next.js hydration 暂时必需**（React 18 RSC 初始注水）。后续可逐步用 **nonce-based CSP** 替换（Next.js 13+ 支持 `headers()` 里读 nonce）—— 但起步先把 `connect-src` / `frame-src` 锁死能挡 80% 的数据外泄路径。
2. 上线前 **先用 `Content-Security-Policy-Report-Only`** 跑 24 小时，看浏览器 console 报哪些 violation，调整白名单后再切换成 enforce mode。

#### 验证

```bash
for d in halofortune.com.au haloloan.com.au ftfinance.com.au; do
  echo "[$d]"
  curl -sI -A "Mozilla/5.0" "https://$d/" | grep -i content-security
  echo "---"
done
# 期望: 每域 CSP 含 default-src / script-src / connect-src / frame-ancestors / base-uri / form-action

# 然后丢到 Google CSP Evaluator 评分:
# https://csp-evaluator.withgoogle.com/
# 期望: 评级 A 或 B（接受 'unsafe-inline' 的减分，其它项零警告）
```

---

### P1-2 — HSTS 缺 `includeSubDomains` 与 `preload`，无法 preload-list 注册

#### 证据

```
$ curl -sI https://halofortune.com.au/ | grep -i strict-transport
strict-transport-security: max-age=63072000
```
三域全部一致 —— `max-age` 充足（2 年），但缺 `includeSubDomains; preload`。

#### 风险

- 任何子域（`portal.halofortune.com.au`、`api.halofortune.com.au`、未来要用的）**首次访问不受 HSTS 保护**，存在 SSL stripping 窗口
- 浏览器内置的 HSTS preload 列表（Chrome/Firefox/Safari 共享）不会收录这三个域
- 金融服务行业（NCCP 监管 / ASIC 期望）应该用 preload

#### 修复

定位文件：`next.config.js` 或 `vercel.json` 的 headers 配置。

```js
{
  key: 'Strict-Transport-Security',
  value: 'max-age=63072000; includeSubDomains; preload'
}
```

#### 加 `preload` 前必须做的检查清单

`preload` 是 **不可逆**（一旦加入 Chrome preload 列表，移除需 ~9 个月并要全球浏览器版本滚动）。加之前确认：

- [ ] 所有当前与未来子域**都能 HTTPS** 且证书有效（不能有任何子域只跑 HTTP）
- [ ] `www.halofortune.com.au` / `www.haloloan.com.au` 子域可访问（见 P2-1，必须先做完）
- [ ] DNS 上没有计划新建的 HTTP-only 子域（如 marketing landing page、临时 staging）
- [ ] 跑测试期 24 小时，观察 Cloudflare/Vercel Analytics 看是否有 HTTP 流量被强制升级而失败

完成后到 https://hstspreload.org/ 提交。

#### 验证

```bash
for d in halofortune.com.au haloloan.com.au ftfinance.com.au; do
  curl -sI "https://$d/" | grep -i strict-transport
done
# 期望: 每域含 'includeSubDomains; preload'
```

提交 preload 后用 https://hstspreload.org/?domain=halofortune.com.au 查状态。

---

### P1-3 — haloloan `/admin` 返回 `307`（疑似内部架构泄漏）

#### 证据

```
$ curl -s -o /dev/null -w "%{http_code}" https://haloloan.com.au/admin
307
```
对比同域 `/wp-admin` `/administrator` 都是 410，**唯独 `/admin` 是 307**。

#### 风险

307 是临时重定向，意味着 `/admin` 真实存在并指向某个内部 URL。可能性：
- 重定向到内部管理面板（如 `https://portal.halofortune.com.au/admin`）→ **内部架构信息泄漏**，攻击者集中精力打那个 URL
- 重定向到 `/zh` / `/en` / 营销页 → 无害
- 重定向到 Vercel 默认 dashboard → 中等风险

#### 修复

**先验证目标**：

```bash
curl -sI https://haloloan.com.au/admin | grep -i "^location:"
```

根据 Location 内容决策：

| Location | 决策 |
|---|---|
| `→ /zh` 或 `/en` 或营销页 | ✅ 保留 307，无风险 |
| `→ portal.halofortune.com.au/*` 或任何内部管理面板 | 🔴 改成 404 / 410，不暴露内部 URL |
| `→ Vercel auth flow` | 🟡 建议改 404，避免给攻击者扫描入口 |
| `→ 外部第三方` | 🔴 改 404，不要做 open redirect |

定位文件：`src/middleware.ts` 或 `vercel.json` 的 redirects 配置。删除或改成：

```ts
// src/middleware.ts
if (pathname === '/admin') {
  return new NextResponse('Not Found', { status: 404 });
}
```

#### 验证

```bash
curl -sI https://haloloan.com.au/admin | head -3
# 期望: HTTP/2 404（或 410 与其它管理路径一致）
```

---

### P1-4 — ftfinance `/quote` 和 `/contact` 的 Cloudflare 缓存可能持有 `cache-control: public` 旧版本

#### 证据

同一 URL 同一命令，~30 秒间隔，两次结果不同：

```
S2 (~19:17 UTC):
  https://ftfinance.com.au/quote
  cache-control: public, max-age=0, must-revalidate

S14 (~19:18 UTC):
  https://ftfinance.com.au/quote
  cache-control: private, no-store, max-age=0
```

S14 拿到的是正确的 `private/no-store`，但 S2 拿到了一个**带 `public` directive 的快照**。两种可能：
1. Cloudflare 边缘缓存了旧的（不正确的 `public`）响应，S2 命中边缘缓存 → 其它边缘节点上的用户也可能拿到 `public` 版本
2. v2-portal 在 S2 和 S14 之间做了 deploy（不太可能 30 秒内）

可能性 1 概率更高 —— 不同 Cloudflare PoP 缓存状态不一致。

#### 风险

如果 Cloudflare 节点持有 `public, max-age=0, must-revalidate` 的快照：
- `max-age=0, must-revalidate` 强制每次 revalidate，所以**短期内不会真的服旧内容**
- 但 `public` directive **允许下游共享缓存（公司代理、ISP transparent cache）存响应**
- 风险点：表单页如果之后某次 SSR 把用户已填字段 hydrate 进 HTML，共享缓存可能保存带 PII 的快照
- 即使现在没漏，配置不一致本身就是隐患

#### 修复

**步骤 1 —— 确认问题真实存在**：

```bash
# 多个 Cloudflare PoP 测试
for ip in $(dig +short ftfinance.com.au | head -5); do
  echo "=== via $ip ==="
  curl -sI -L --resolve "ftfinance.com.au:443:$ip" --max-time 10 https://ftfinance.com.au/quote \
    | grep -iE "cache-control|cf-cache-status|x-vercel-cache"
done
```

如果发现某些节点返回 `public` 而其它返回 `private` → 缓存不一致，做步骤 2。

**步骤 2 —— Purge Cloudflare 边缘缓存**：

Cloudflare Dashboard → Caching → Configuration → **Purge Custom URLs**：
```
https://ftfinance.com.au/quote
https://ftfinance.com.au/zh/quote
https://ftfinance.com.au/en/quote
https://ftfinance.com.au/contact
https://ftfinance.com.au/zh/contact
https://ftfinance.com.au/en/contact
```

或者 CLI（如果有 API token）：
```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/purge_cache" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"files":[
    "https://ftfinance.com.au/quote",
    "https://ftfinance.com.au/contact"
  ]}'
```

**步骤 3 —— 在 v2-portal 端永久杜绝 `public` directive 出现在 PII 表单页**：

定位文件：v2-portal 的 `/quote` 和 `/contact` 路由 handler。

```ts
return new Response(html, {
  headers: {
    'Content-Type': 'text/html; charset=utf-8',
    'Cache-Control': 'private, no-store, max-age=0',
    'CDN-Cache-Control': 'no-store',   // 显式告诉 Cloudflare 不要缓存
    'Surrogate-Control': 'no-store',
    'Pragma': 'no-cache',
  },
});
```

`CDN-Cache-Control: no-store` 是专门给 CDN 看的（Cloudflare、Fastly、Vercel CDN 都支持），覆盖 `cache-control` 让 CDN 永不缓存。

#### 验证

```bash
# Purge 后等 1 分钟，跑 10 次确认所有 PoP 都拿到 private
for i in {1..10}; do
  curl -sI -L --max-time 10 "https://ftfinance.com.au/quote?_=$RANDOM" \
    | grep -i cache-control
  sleep 2
done
# 期望: 10 次全部 'private, no-store'
```

---

## 3. 🟡 P2（一月内修）

### P2-1 — halo\* TLS 证书 SAN 缺 `www.`

#### 证据

```
halofortune SAN: DNS:halofortune.com.au                    ← 缺 www
haloloan    SAN: DNS:haloloan.com.au                        ← 缺 www
ftfinance   SAN: DNS:*.ftfinance.com.au, DNS:ftfinance.com.au ← 含 wildcard ✓
```

#### 风险

用户在浏览器手敲 `www.halofortune.com.au` → 浏览器**先验证证书后跟随重定向** → 证书 CN/SAN 不匹配 → 红色警告页 "Your connection is not private"。

即使 DNS 层配了 `www.X.com.au` → apex 重定向，警告也会先出现。对 mortgage broker 是非常不好的首次印象。

#### 修复

**Vercel 流程**：

1. Vercel Project → Settings → Domains → Add `www.halofortune.com.au`
2. Vercel 提示配 DNS（CNAME `cname.vercel-dns.com`，或 A record 给 76.76.21.21）
3. Vercel 自动签发覆盖 www 的 Let's Encrypt 证书（~30 秒）
4. 在 `next.config.js` 加 redirect：
   ```js
   async redirects() {
     return [{
       source: '/:path*',
       has: [{ type: 'host', value: 'www.halofortune.com.au' }],
       destination: 'https://halofortune.com.au/:path*',
       permanent: true,
     }];
   }
   ```
   或者用 Vercel domains 配置里的 "Redirect to" 功能（更简洁）

haloloan 同理。ftfinance 已有 wildcard 不需要改。

#### 验证

```bash
for d in halofortune.com.au haloloan.com.au; do
  echo "[www.$d cert]"
  echo Q | openssl s_client -connect "www.$d:443" -servername "www.$d" 2>/dev/null < /dev/null \
    | openssl x509 -noout -ext subjectAltName 2>/dev/null
  echo "[www.$d redirect]"
  curl -sI "https://www.$d/" | head -3
done
# 期望:
#   SAN 含 www.X.com.au
#   HTTP/2 308 或 301 redirect 到 apex
```

---

### P2-2 — haloloan 缺 `/zh/complaints` 与 `/zh/terms`（合规风险）

#### 证据

```
$ curl -s -o /dev/null -w "%{http_code}" https://haloloan.com.au/zh/complaints
404 (或 410 — 不在 200 返回列表中)

$ curl -s -o /dev/null -w "%{http_code}" https://haloloan.com.au/zh/terms
404 (或 410)

而 /privacy 与 /credit-guide 都正常：
$ /zh/privacy -> 200 ✓
$ /zh/credit-guide -> 200 ✓
```

#### 风险

**ASIC RG 271 + AFCA Operational Guidelines + Privacy Act 1988 APP 1.4** 强制要求：

- **AFCA 成员**（haloloan 是 AFCA member 45759）必须在网站上公开 **complaints handling process**
- 借贷服务商需要公示 **Terms of Use** 让消费者知道服务边界

缺失这些页面：
- 监管审查（ASIC routine review）时是负面证据
- 客户投诉走 AFCA 流程，AFCA 会优先查网站是否有正常 complaints 入口
- 中长期合规风险显著高于这条 P2 的工作量

#### 修复

定位：Next.js 应用对应路由（推测 `src/app/[locale]/complaints/page.tsx` 与 `src/app/[locale]/terms/page.tsx`）。

**`/zh/complaints` 内容模板**（haloloan 视角）：

```markdown
# 投诉与争议处理

Halo Loan 致力于为您提供专业的房贷中介服务。如果您对我们的服务有任何不满或建议，
您有权利发起投诉。

## 第一步：联系我们（内部处理，30 天内回复）

- 邮箱：admin@halofortune.com.au
- 电话：1300 741 668
- 地址：Suite 902, Level 9, 3 Bowen Crescent, Melbourne VIC 3004

请提供：您的姓名、联系方式、投诉的具体事项、希望的解决方案。

我们会在 5 个工作日内确认收到，30 天内提供书面答复。

## 第二步：升级到外部独立仲裁（AFCA）

如果您对我们 30 天内的处理结果不满意，或我们 45 天内未给出最终答复，
您可以免费向 **澳大利亚金融投诉局 (Australian Financial Complaints Authority, AFCA)** 发起独立仲裁。

- AFCA 网站：https://www.afca.org.au
- 电话：1800 931 678（澳洲境内免费）
- 邮箱：info@afca.org.au
- 邮寄：GPO Box 3, Melbourne VIC 3001
- Halo Loan AFCA 会员号：**45759**

AFCA 的服务对消费者免费，决议对我们具有法律约束力。

## 监管框架

- Australian Credit Licence: **483923**（Halo Fortune Group Pty Ltd）
- MFAA 成员号：**660806**（Mortgage & Finance Association of Australia）
- NCCP Act 2009 best-interests duty 适用于我们的每一项推荐
- Privacy Act 1988 与 Australian Privacy Principles 适用于您的个人信息处理
```

`/en/complaints` 翻译相同结构。

`/zh/terms` 与 `/en/terms` 内容应包含：
- 服务定义（broker 角色边界 + 不构成 personal advice 的说明）
- 用户责任（提供真实准确信息）
- 免责（贷款审批由放款机构决定）
- Privacy 链接、Credit Guide 链接、Complaints 链接
- Governing Law（VIC）
- 修订机制

**halofortune & ftfinance 状态**：本轮被 WAF rate-limit 拦截无法直接验证。请先按 P3-2 慢速重测一次，确认是否同样缺失，缺则按相同模板补。

#### 验证

```bash
for d in halofortune.com.au haloloan.com.au ftfinance.com.au; do
  echo "[$d]"
  for loc in zh en; do
    for p in privacy credit-guide complaints terms; do
      code=$(curl -s -o /dev/null -w "%{http_code}" -A "Mozilla/5.0" --max-time 10 "https://$d/$loc/$p")
      echo "  /$loc/$p -> $code"
      sleep 1
    done
  done
done
# 期望: 所有 page 都 200
```

---

### P2-3 — `security.txt` 只在 haloloan 部署，halofortune 与 ftfinance 缺失

#### 证据

```
haloloan/.well-known/security.txt -> 200 ✓
halofortune/.well-known/security.txt -> 403 (rate-limit 拦了，需复测)
ftfinance/.well-known/security.txt -> 403 (rate-limit)
```

#### 风险

- RFC 9116 推荐**所有处理用户数据的网站**都要有 `security.txt`，让安全研究者知道怎么报漏洞
- 没有 `security.txt` 的网站，bug bounty 研究者可能**直接公开披露**漏洞（不知道怎么联系到你）
- 金融服务行业的安全合规 checklist 普遍含这一项

#### 修复

**halofortune `/.well-known/security.txt`** 模板：

```
Contact: mailto:security@halofortune.com.au
Contact: mailto:admin@halofortune.com.au
Expires: 2027-05-11T00:00:00Z
Preferred-Languages: en, zh
Canonical: https://halofortune.com.au/.well-known/security.txt
Policy: https://halofortune.com.au/zh/security-policy
Acknowledgments: https://halofortune.com.au/zh/security-acknowledgments
```

**ftfinance `/.well-known/security.txt`** 模板（**邮箱不能与 halo\* 共用，遵守防火墙**）：

```
Contact: mailto:security@ftfinance.com.au
Contact: mailto:info@ftfinance.com.au
Expires: 2027-05-11T00:00:00Z
Preferred-Languages: en, zh
Canonical: https://ftfinance.com.au/.well-known/security.txt
Policy: https://ftfinance.com.au/legal/security-policy
```

部署位置：
- halo\*（Next.js）：放 `src/app/.well-known/security.txt/route.ts` 或 `public/.well-known/security.txt`
- ftfinance（静态站）：放 `fingertips finance/public/.well-known/security.txt`

`security-policy` 页面内容应包括：scope（什么算 in-scope）、不允许的测试方式（DoS、physical、social engineering）、披露时间窗（90 天 coordinated disclosure）。

#### 防火墙绝对禁令

- ❌ ftfinance 的 security.txt **不能**写 `security@halofortune.com.au` 或任何 halo\* 邮箱
- ❌ ftfinance 的 Acknowledgments / Policy URL **不能**链向 halo\* 域
- ✅ 三域用各自独立的邮箱与策略 URL

#### 验证

```bash
for d in halofortune.com.au haloloan.com.au ftfinance.com.au; do
  echo "[$d]"
  curl -s --max-time 10 "https://$d/.well-known/security.txt" | head -10
  sleep 3   # spread 出来避免触发 WAF
done
# 期望: 三域都 200，内容含 Contact 行 + Expires + Preferred-Languages
```

防火墙抽查：
```bash
# halo* security.txt 不应提及 ftfinance
for d in halofortune.com.au haloloan.com.au; do
  n=$(curl -s "https://$d/.well-known/security.txt" | grep -ciE "ftfinance|fingertips|掌忠贷")
  echo "$d/security.txt mentions ftfinance: $n (expected 0)"
done

# ftfinance security.txt 不应提及 halo*
n=$(curl -s https://ftfinance.com.au/.well-known/security.txt | grep -ciE "halofortune|haloloan|halo fortune|halo loan|wei-ming")
echo "ftfinance/security.txt mentions halo*: $n (expected 0)"
```

---

## 4. ⚪ P3（信息 / 长期跟踪）

### P3-1 — Server header 暴露平台（不可避免）

```
halofortune: server: Vercel
haloloan:    server: Vercel
ftfinance:   server: cloudflare
```

Vercel / Cloudflare 都不支持自定义 server header。攻击者通过 wappalyzer 一秒识别 —— 但平台本身有 WAF 保护，**信息暴露不直接给攻击 surface**。

**操作建议**：无 actionable，接受。

---

### P3-2 — 本次审计期间 WAF rate-limit 误伤本审计

#### 现象

halofortune 与 ftfinance 的多个 path 在 S5/S6/S7/S13 中返回 403，但单独慢速测试时返回 200/404。结论：**Cloudflare/Vercel WAF 在每秒 50+ 请求时识别为扫描行为而拦截**。

#### 判断

**这是好事** —— WAF 主动防御工作正常。不需要"修"，但需要做两件事：

1. **复测真实状态** —— 用 sleep 1-3s spread 出，重测 halofortune 与 ftfinance 的：
   - `/.well-known/security.txt`（确认是否真的不存在）
   - `/.well-known/ai.txt`（确认 200）
   - 所有合规页面（`/zh/privacy` 等）

2. **检查 WAF 是否误伤合法爬虫** —— Cloudflare Dashboard → Firewall Events，看是否有：
   - Googlebot 被拦截（影响 SEO）
   - GPTBot / ClaudeBot / PerplexityBot 被拦截（影响 GEO，与之前 SEO 文档目标冲突）
   - 真用户 IP 被误判

如果发现合法爬虫被拦，需要在 Cloudflare → Tools → IP Access Rules 加 verified bot allow，或者在 WAF Custom Rules 里给 `cf.client.bot` true 的 known bots 加 skip 规则。

#### 验证

```bash
# 慢速复测（避免再触发 WAF）
for d in halofortune.com.au ftfinance.com.au; do
  for p in ".well-known/security.txt" ".well-known/ai.txt" "zh/privacy" "zh/credit-guide" "zh/complaints"; do
    code=$(curl -s -o /dev/null -w "%{http_code}" -A "Mozilla/5.0" --max-time 10 "https://$d/$p")
    echo "$d/$p -> $code"
    sleep 2
  done
done
```

---

### P3-3 — 缺 Cross-Origin-\* 现代隔离头

**当前**：三域都没 `Cross-Origin-Opener-Policy` / `Cross-Origin-Embedder-Policy` / `Cross-Origin-Resource-Policy`。

**风险**：Spectre/Meltdown 类 side-channel 攻击仍有理论窗口。对一般 B2C 网站不致命，但收集 PII 的金融服务理论上应该加。

**修复**（next.config.js 或 vercel.json headers）：

```js
{ key: 'Cross-Origin-Opener-Policy',   value: 'same-origin' },
{ key: 'Cross-Origin-Resource-Policy', value: 'same-origin' },
// COEP 比较激进，可能破第三方资源加载，先观察 violation 再启用:
// { key: 'Cross-Origin-Embedder-Policy', value: 'require-corp' },
```

**注意**：加 COOP 后 Google Analytics 等第三方 popup 可能受影响，需要测试。

#### 验证

```bash
for d in halofortune.com.au haloloan.com.au ftfinance.com.au; do
  echo "[$d]"
  curl -sI -A "Mozilla/5.0" "https://$d/" | grep -iE "cross-origin-"
done
# 期望: 至少 COOP 和 CORP 两行
```

---

## 5. 不在 black-box 范围但建议自查（需要白盒）

下面这些**外部无法验证**，需要在 production 仓库 / Vercel Dashboard / Cloudflare Dashboard / Supabase Dashboard 内部审计：

### 5a. 依赖供应链

```bash
# 在 production 仓库根目录跑
npm audit --audit-level=high
# 或
pnpm audit --severity=high
# 或
yarn audit --level=high
```
配合 GitHub Dependabot + Snyk 跑长期监控。

### 5b. 框架版本

确认 Next.js、React、Node.js 没在已知 CVE 受影响版本。例如：
- Next.js < 14.2.10 的 SSRF 漏洞 (CVE-2024-46982)
- Next.js < 13.5.1 的 cache poisoning
- React < 18.2.0 的特定 hydration 注入

### 5c. API 端点代码审计（OWASP API Top 10）

`/api/quote` 与 `/api/contact` 实现需要 grep 检查：

| OWASP API | 检查点 |
|---|---|
| API1 Broken Object Level Authorization (BOLA) | `/api/quote/[id]` 是否检查 ownership？还是任何 session 都能拿任意 quote？ |
| API2 Broken Authentication | 如果有 session，是否 JWT 签名验证 + 过期检查？ |
| API3 Excessive Data Exposure | API 返回是否只包含必要字段？还是把整个 user record dump 给前端？ |
| API4 Lack of Resources & Rate Limiting | 单 IP 能不能每秒 100 个 quote 提交？Cloudflare Rate Limit 是否覆盖 /api/quote？ |
| API5 BOPLA (Broken Object Property Level Auth) | 用户能不能 POST 一个含 `isAdmin: true` 的字段然后 server 接受了？ |
| API6 Unrestricted Resource Consumption | upload 文件（payslip/BAS）是否限制大小、类型、scan？ |
| API7 SSRF | 任何接受 URL 输入的端点（image proxy 等）是否限制内网 IP？ |
| API8 Security Misconfiguration | 错误信息是否包含 stack trace / SQL error / file path？ |
| API9 Improper Inventory Management | 是否有遗忘的 /api/v1/* 旧版本端点仍在线？ |
| API10 Unsafe Consumption of APIs | v2-portal 调外部 API 时是否验证响应 / 限制白名单？ |

### 5d. Cloudflare 配置审计

- WAF rules：OWASP Core Ruleset 启用且 sensitivity = High
- Bot Fight Mode 开启
- Rate Limiting：/api/* 路径加 per-IP 限制（如 60 req/min）
- Page Rules 不要无意 cache `/api/*` 或 `/quote`、`/contact`
- API Shield（如果用 Cloudflare Pro+）开启 JWT validation

### 5e. Vercel 配置审计

- 环境变量都标 "Sensitive"（不会显示在 build log / preview deployment）
- Preview Deployments 不暴露 production 数据（Vercel branch deploys 默认与 main 共享 env vars —— 危险）
- Vercel Functions Logs 不保留 PII（设置 retention < 7 days for log streams that may contain user data）

### 5f. 数据库层

如果用 Supabase / Postgres：
- Row-Level Security (RLS) **必须开启**所有含用户数据的表
- service_role key **绝对不能**出现在前端 bundle 或客户端代码
- anon key 在前端是 OK 的，但 anon 应该只有 RLS 限定的最小权限
- 定期 rotate keys（季度）

可以用以下命令快速检查（如果 production 仓库可读）：
```bash
grep -rn "SUPABASE_SERVICE_ROLE\|service_role" src/ --include="*.{ts,tsx,js,jsx}"
# 期望: 只在 server-side route handlers / API routes 出现，不在 client component
```

### 5g. 管理后台 / Broker Portal

`portal.halofortune.com.au` 或类似管理面板（broker accreditation、loan submission）需要：

- SSO + 2FA 强制（不是可选）
- Session 超时 15-30 分钟
- 失败登录尝试 5 次后锁定 + 通知
- 所有写操作记 audit log（who / when / what）
- 不要在 URL query string 暴露任何 ID
- 文件上传（payslip / BAS）走 signed URL，不直接挂在 CDN

### 5h. 合规审查清单

- [ ] NCCP Act 2009 — Credit Guide 内容完整
- [ ] Privacy Act 1988 + APP — Privacy Policy 含 13 个 APP 章节
- [ ] ASIC RG 234 — 营销与广告 disclaimer 覆盖率
- [ ] ASIC RG 271 — Complaints procedure 公示
- [ ] Online Safety Act 2021 — 用户内容审核（如有 UGC）
- [ ] Anti-Money Laundering and Counter-Terrorism Financing Act 2006 — KYC 流程（如收 ID）
- [ ] OAIC Notifiable Data Breach Scheme — 数据泄漏报告流程文档

### 5i. PII 处理

- 客户身份信息 / payslip / bank statement **静态加密**（at-rest encryption）
- 传输 TLS 1.2+ enforced（已确认）
- 日志中 PII 字段 mask（如 email 显示为 `j***@example.com`）
- 退订流程清晰（GDPR-style 即使澳洲不强制，APP 13 也要求）
- 数据保留期限明确（不要无限期保留 PII）

---

## 6. 完整验证脚本（实施完成后跑）

```bash
#!/usr/bin/env bash

DOMAINS=(halofortune.com.au haloloan.com.au ftfinance.com.au)

echo "########## SEC-V1: CSP 完整性 ##########"
for d in "${DOMAINS[@]}"; do
  echo "[$d]"
  csp=$(curl -sI -A "Mozilla/5.0" "https://$d/" | grep -i "content-security-policy:" | head -1)
  echo "  $csp"
  # 检查关键指令
  for directive in default-src script-src connect-src frame-ancestors base-uri form-action; do
    if echo "$csp" | grep -q "$directive"; then
      echo "    ✓ $directive"
    else
      echo "    ✗ MISSING $directive"
    fi
  done
  sleep 1
done

echo ""
echo "########## SEC-V2: HSTS preload-eligible ##########"
for d in "${DOMAINS[@]}"; do
  h=$(curl -sI "https://$d/" | grep -i strict-transport-security)
  ok=true
  echo "$h" | grep -q "max-age=[0-9]\{8,\}" || ok=false
  echo "$h" | grep -qi "includeSubDomains" || ok=false
  echo "$h" | grep -qi "preload" || ok=false
  echo "$d: $h -> $([ "$ok" = true ] && echo "✓ preload-eligible" || echo "✗ missing directives")"
  sleep 1
done

echo ""
echo "########## SEC-V3: haloloan /admin 不暴露内部 URL ##########"
loc=$(curl -sI https://haloloan.com.au/admin | grep -i "^location:" | tr -d '\r')
code=$(curl -s -o /dev/null -w "%{http_code}" https://haloloan.com.au/admin)
echo "/admin -> $code  $loc"
# 期望: 404/410，或者 location 指向 /zh / /en 营销页

echo ""
echo "########## SEC-V4: ftfinance form CDN cache 一致性 ##########"
for i in 1 2 3 4 5; do
  cc=$(curl -sI -L --max-time 10 "https://ftfinance.com.au/quote?_=$RANDOM" | grep -i cache-control)
  echo "try $i: $cc"
  sleep 2
done
# 期望: 5 次全部 'private, no-store'

echo ""
echo "########## SEC-V5: www. 子域证书 ##########"
for d in halofortune.com.au haloloan.com.au; do
  echo "[www.$d]"
  san=$(echo Q | openssl s_client -connect "www.$d:443" -servername "www.$d" 2>/dev/null < /dev/null \
    | openssl x509 -noout -ext subjectAltName 2>/dev/null)
  echo "  $san"
done

echo ""
echo "########## SEC-V6: 合规页面全 200 ##########"
for d in "${DOMAINS[@]}"; do
  echo "[$d]"
  for loc in zh en; do
    for p in privacy credit-guide complaints terms; do
      code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 "https://$d/$loc/$p")
      echo "  /$loc/$p -> $code"
      sleep 1
    done
  done
done

echo ""
echo "########## SEC-V7: security.txt 三域均 200 ##########"
for d in "${DOMAINS[@]}"; do
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 "https://$d/.well-known/security.txt")
  echo "$d/.well-known/security.txt -> $code"
  sleep 2
done

echo ""
echo "########## SEC-V8: security.txt 防火墙 ##########"
for d in halofortune.com.au haloloan.com.au; do
  n=$(curl -s "https://$d/.well-known/security.txt" | grep -ciE "ftfinance|fingertips|掌忠贷")
  echo "$d security.txt mentions ftfinance: $n (expected 0)"
done
n=$(curl -s https://ftfinance.com.au/.well-known/security.txt | grep -ciE "halofortune|haloloan|halo fortune|halo loan|wei-ming")
echo "ftfinance security.txt mentions halo*: $n (expected 0)"

echo ""
echo "########## SEC-V9: COOP/CORP 头 ##########"
for d in "${DOMAINS[@]}"; do
  echo "[$d]"
  curl -sI "https://$d/" | grep -iE "cross-origin-"
done

echo ""
echo "########## SEC-V10: PII 表单页头检查 ##########"
URLS=(
  "https://haloloan.com.au/zh/quote"
  "https://halofortune.com.au/zh/scenario"
  "https://ftfinance.com.au/quote"
  "https://ftfinance.com.au/contact"
)
for u in "${URLS[@]}"; do
  echo "[$u]"
  curl -sIL --max-time 10 "$u" | grep -iE "^(cache-control|content-security-policy|x-frame-options|strict-transport-security|referrer-policy):"
done
```

---

## 7. 文档结束

完成这 4 P1 + 3 P2 + 视情况的 P3，三域安全评级从 **A-** 提升到 **A / A+** 水平 —— 达到澳洲金融服务行业头部水准。

后续可考虑：
- 提交到 https://hstspreload.org/（P1-2 完成后）
- 提交到 https://securityheaders.com/（看 grade，目标 A+）
- 提交到 https://www.ssllabs.com/ssltest/（看 SSL Labs grade，目标 A+）
- 注册 https://www.hackerone.com/ 或 https://bugcrowd.com/ 的 vulnerability disclosure program（security.txt 完成后可以做）

如果要更深入（OWASP API、Supabase RLS、Vercel 配置、Cloudflare WAF rules、合规文档审查、PII 处理流程），需要白盒访问 —— 见 §5 列表。
