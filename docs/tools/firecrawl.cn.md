---
summary: "Firecrawl fallback for web_fetch (anti-bot + cached extraction)"
read_when:
  - You want Firecrawl-backed web extraction
  - You need a Firecrawl API key
  - You want anti-bot extraction for web_fetch
---

# Firecrawl

Clawdbot 可以使用 **Firecrawl** 作为 `web_fetch` 的回退提取器。它是一个托管内容提取服务，支持机器人规避和缓存，这有助于处理 JS 密集型站点或阻止纯 HTTP 获取的页面。

## 获取 API 密钥

1) 创建 Firecrawl 账户并生成 API 密钥。
2) 将其存储在配置中或在网关环境中设置 `FIRECRAWL_API_KEY`。

## 配置 Firecrawl

```json5
{
  tools: {
    web: {
      fetch: {
        firecrawl: {
          apiKey: "FIRECRAWL_API_KEY_HERE",
          baseUrl: "https://api.firecrawl.dev",
          onlyMainContent: true,
          maxAgeMs: 172800000,
          timeoutSeconds: 60
        }
      }
    }
  }
}
```

注意：
- 当存在 API 密钥时，`firecrawl.enabled` 默认为 true。
- `maxAgeMs` 控制缓存结果可以有多旧（毫秒）。默认是 2 天。

## 隐身 / 机器人规避

Firecrawl 公开**代理模式**参数用于机器人规避（`basic`、`stealth` 或 `auto`）。
Clawdbot 始终对 Firecrawl 请求使用 `proxy: "auto"` 加上 `storeInCache: true`。
如果省略代理，Firecrawl 默认为 `auto`。`auto` 在基本尝试失败时使用隐身代理重试，这可能比仅基本抓取使用更多积分。

## `web_fetch` 如何使用 Firecrawl

`web_fetch` 提取顺序：
1) Readability（本地）
2) Firecrawl（如果配置）
3) 基本 HTML 清理（最后回退）

有关完整的 Web 工具设置，请参见 [Web 工具](/tools/web）。
