# Neko歌姬计划 · 防爬与客户端识别（专项）

本文档描述 Neko歌姬计划后端对 JSON 接口（`/api/*`）的**防爬与客户端区分拦截**策略：
如何拦截已知爬虫/安全扫描器，以及如何区分「未知 / 小众网站爬虫」与「真实浏览器、应用内 WebView、原生客户端」。
所有接口的通用响应契约见主文档 [README.md](README.md)。

> 目标：主要拦截**未知网站、小众网站**的爬虫与探测工具——这类 UA 往往不含常见关键词（`bot`/`crawler`），
> 甚至只伪造一个 `Mozilla/5.0` 前缀。仅靠关键词黑名单会大面积漏网，因此引入「浏览器完整性」正向校验。

---

## 1. 拦截模型（两段式）

对 `/api/*` 的每个请求，按下列顺序判定：

| 顺序 | 判定 | 命中动作 |
| --- | --- | --- |
| 0 | 路径为 `/api/payment/zpay/notify` | **直接放行**（支付平台回调，豁免防爬与限流） |
| 1 | UA 命中**已知爬虫 / 无头 / 命令行 / 安全扫描器**关键词 | **403** |
| 2 | UA 命中**放行名单**（`network.allow_client_user_agents`） | 放行 |
| 3 | UA 为**内置原生客户端**（Android okhttp/Dalvik、PC Qt/Electron、播放器 libmpv/VLC/FFmpeg 等） | 放行 |
| 4 | UA **结构像真浏览器** 且请求带**浏览器特征头** | 放行 |
| 5 | 其余（未知爬虫、残缺/仅伪造 UA、空 UA、扫描器） | **403** |

第 2~5 步属于**浏览器完整性区分拦截**，可用 `network.browser_integrity_enabled=false` 关闭（关闭后退化为仅第 1 步黑名单）。

### 1.1 已知爬虫 / 扫描器关键词（第 1 步）

覆盖搜索引擎、AI 抓取、链接预览、SEO 分析、命令行客户端、无头浏览器、监控归档，以及常见安全扫描器
（`sqlmap`、`nikto`、`nmap`、`masscan`、`zgrab`、`nuclei`、`wpscan`、`gobuster`、`ffuf`、`feroxbuster`、
`acunetix`、`nessus`、`openvas`、`burpsuite`、`zaproxy`、`whatweb`、`w3af`、`arachni`、`skipfish`、
`jaeles`、`commix`、`dalfox`、`wapiti`、`netsparker`、`sslscan`、`sslyze`、`testssl` 等）。

### 1.2 浏览器结构判定（第 4 步其一）

UA 必须同时满足：

- 含 `Mozilla/`；
- 含浏览器内核标记之一：`AppleWebKit/`（Chrome / Edge / Opera / Safari / iOS·Android WebView）、
  `Gecko/`（Firefox）、`Trident/`（旧 Edge / IE11）。

> 注意：iOS `WKWebView`（微信、QQ、支付宝等应用内浏览器）的 UA **常省略 `Safari/`、`Version/`**，
> 因此本校验**不强制** Safari 版本串，只要求存在内核标记，避免误伤应用内浏览器。

**会被判定为「非浏览器」的典型 UA**（即使含 `Mozilla/`）：

- `Mozilla/5.0`（裸前缀）
- `Mozilla/5.0 (X11; Linux x86_64)`（无内核标记）
- `Mozilla/5.0 (compatible; AcmeIndex/1.0)`（未知爬虫）
- `Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1)`（无内核标记）

### 1.3 浏览器特征头（第 4 步其二）

请求必须带 `Accept`，且带 `Accept-Language` 或任一 `Sec-Fetch-*`（`Sec-Fetch-Mode` / `Sec-Fetch-Site` / `Sec-Fetch-Dest`）。
真实浏览器（含媒体 `<audio>` 请求，通常带 `Accept: */*` + `Accept-Language` + `Sec-Fetch-Dest: audio`）恒满足。

> 仅伪造浏览器 UA、但不带浏览器头的脚本会被第 4 步拒绝。
> 说明：请求头可被有能力的攻击者一并伪造，本策略用于显著提高门槛、拦截绝大多数未知/小众爬虫；对高度定制的爬虫属于持续对抗。

### 1.4 内置原生客户端白名单（第 3 步）

UA 含下列标记之一即放行：`okhttp`、`dalvik`、`libmpv`、`mpv/`、`vlc`（前缀）、`ffmpeg`、`ffprobe`、
`android`、`electron`、`qt`（前缀）、`qts`、`qtwebengine`。

---

## 2. 命中响应

被拦截的请求返回 **403**，响应体与全局错误契约一致：

```json
{
  "success": false,
  "message": "请求已拒绝",
  "data": null
}
```

响应头固定为 `Cache-Control: private, no-store`。

---

## 3. 配置项

见 `backend/src/main/resources/config.yml` 的 `network` 段：

```yaml
network:
  # 可信客户端 IP 头（影响评论归属地与 IP 限流；详见主文档与部署说明）
  trusted_client_ip_header: X-Real-IP

  # 总开关：关闭后 /api 不再做任何防爬拦截
  crawler_protection_enabled: true

  # 浏览器完整性区分拦截（拦截未知/小众爬虫与安全扫描器）
  # false = 仅保留关键词黑名单
  browser_integrity_enabled: true

  # 额外放行的客户端 UA 子串（大小写不敏感），用于登记内置白名单之外的第三方客户端
  allow_client_user_agents: []
```

IP 频率限制见 `rate_limit` 段（按 /24 聚合计数与封锁，超限返回 429 与 `Retry-After`）。

---

## 4. 客户端接入建议

- **浏览器 / Web 前端**：无需任何改动（真实浏览器天然满足结构与特征头校验）。
- **应用内 WebView（微信 / QQ / 支付宝等）**：无需改动（UA 含 `AppleWebKit/`，且带浏览器特征头）。
- **Android / PC 原生客户端**：使用内置白名单内的 UA（如 `okhttp`、`dalvik`、`qt`、`Electron`）即可。
- **第三方客户端 / 脚本集成**：若 UA 不在内置白名单，请使用带版本号的自定义 UA，并加入
  `network.allow_client_user_agents` 登记放行，例如：

  ```yaml
  allow_client_user_agents:
    - "MyThirdPartyClient/"
  ```

---

## 5. 与 SEO 抓取的关系

搜索引擎等需要被收录的抓取器**不应访问 JSON 接口**，而应访问页面（首页、详情页、歌单页等）。
这类请求由 SEO 渲染链路处理（详见主文档相关章节），与本文档的 `/api` 防爬互不影响。

---

## 6. 通用安全加固（方法限制与响应头）

除防爬外，后端对所有响应统一做以下加固（`SecurityHeadersFilter` + Jetty 连接器配置）：

| 项 | 策略 |
| --- | --- |
| `TRACE` / `TRACK` 方法 | 返回 `405 Method Not Allowed`，`Allow` 列出常规方法（防 XST） |
| `Server` 响应头 | 不发送带版本号的实现标识，统一为 `NekoMusic`（避免泄露 Jetty 版本） |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `X-Frame-Options` | `SAMEORIGIN` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` |
| `Strict-Transport-Security` | 由 Nginx / CDN 在 TLS 层下发（见 `deploy/nginx.cdn.conf`），后端明文 HTTP 不下发 |

