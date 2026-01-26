---
summary: "Session pruning: tool-result trimming to reduce context bloat"
read_when:
  - You want to reduce LLM context growth from tool outputs
  - You are tuning agents.defaults.contextPruning
---
# 会话修剪

会话修剪在每次 LLM 调用之前从内存上下文中修剪**旧工具结果**。它**不**重写磁盘上的会话历史（`*.jsonl`）。

## 何时运行
- 当启用 `mode: "cache-ttl"` 且会话的最后一次 Anthropic 调用早于 `ttl` 时。
- 仅影响为该请求发送给模型的消息。
- 仅对 Anthropic API 调用（和 OpenRouter Anthropic 模型）活动。
- 为了获得最佳结果，将 `ttl` 与你的模型 `cacheControlTtl` 匹配。
- 修剪后，TTL 窗口重置，因此后续请求保持缓存，直到 `ttl` 再次过期。

## 智能默认值（Anthropic）
- **OAuth 或 setup-token** 配置文件：启用 `cache-ttl` 修剪并将心跳设置为 `1h`。
- **API 密钥** 配置文件：启用 `cache-ttl` 修剪，将心跳设置为 `30m`，并在 Anthropic 模型上将默认 `cacheControlTtl` 设置为 `1h`。
- 如果你显式设置这些值中的任何一个，Clawdbot **不**覆盖它们。

## 这改善了什么（成本 + 缓存行为）
- **为什么修剪：** Anthropic 提示缓存仅在 TTL 内适用。如果会话在 TTL 后空闲，下一个请求会重新缓存完整提示，除非你先修剪它。
- **什么变得更便宜：** 修剪减少了 TTL 过期后该第一个请求的**cacheWrite**大小。
- **为什么 TTL 重置很重要：** 一旦修剪运行，缓存窗口重置，因此后续请求可以重用新缓存的提示，而不是再次重新缓存完整历史。
- **它不做什么：** 修剪不会添加 tokens 或"加倍"成本；它只更改在该第一个 TTL 后请求上缓存的内容。

## 可以修剪的内容
- 仅 `toolResult` 消息。
- 用户 + 助手消息**永远不**被修改。
- 最后 `keepLastAssistants` 个助手消息受到保护；该截止点之后的工具结果不会被修剪。
- 如果没有足够的助手消息来建立截止点，则跳过修剪。
- 包含**图像块**的工具结果被跳过（永远不修剪/清除）。

## 上下文窗口估计
修剪使用估计的上下文窗口（字符 ≈ tokens × 4）。窗口大小按以下顺序解析：
1) 模型定义 `contextWindow`（来自模型注册表）。
2) `models.providers.*.models[].contextWindow` 覆盖。
3) `agents.defaults.contextTokens`。
4) 默认 `200000` tokens。

## 模式
### cache-ttl
- 仅当最后一次 Anthropic 调用早于 `ttl`（默认 `5m`）时，修剪才运行。
- 运行时：与之前相同的软修剪 + 硬清除行为。

## 软 vs 硬修剪
- **软修剪**：仅用于超大工具结果。
  - 保留头部 + 尾部，插入 `...`，并附加带有原始大小的注释。
  - 跳过带有图像块的结果。
- **硬清除**：用 `hardClear.placeholder` 替换整个工具结果。

## 工具选择
- `tools.allow` / `tools.deny` 支持 `*` 通配符。
- Deny 获胜。
- 匹配不区分大小写。
- 空允许列表 => 允许所有工具。

## 与其他限制的交互
- 内置工具已经截断自己的输出；会话修剪是一个额外的层，可防止长时间运行的聊天在模型上下文中累积太多工具输出。
- 压缩是分开的：压缩总结并持久化，修剪是每个请求的瞬态。参见 [/concepts/compaction](/concepts/compaction)。

## 默认值（启用时）
- `ttl`：`"5m"`
- `keepLastAssistants`：`3`
- `softTrimRatio`：`0.3`
- `hardClearRatio`：`0.5`
- `minPrunableToolChars`：`50000`
- `softTrim`：`{ maxChars: 4000, headChars: 1500, tailChars: 1500 }`
- `hardClear`：`{ enabled: true, placeholder: "[Old tool result content cleared]" }`

## 示例
默认（关闭）：
```json5
{
  agent: {
    contextPruning: { mode: "off" }
  }
}
```

启用 TTL 感知修剪：
```json5
{
  agent: {
    contextPruning: { mode: "cache-ttl", ttl: "5m" }
  }
}
```

限制修剪到特定工具：
```json5
{
  agent: {
    contextPruning: {
      mode: "cache-ttl",
      tools: { allow: ["exec", "read"], deny: ["*image*"] }
    }
  }
}
```

参见配置参考：[Gateway 配置](/gateway/configuration)
