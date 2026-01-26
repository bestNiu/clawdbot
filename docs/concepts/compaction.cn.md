---
summary: "Context window + compaction: how Clawdbot keeps sessions under model limits"
read_when:
  - You want to understand auto-compaction and /compact
  - You are debugging long sessions hitting context limits
---
# 上下文窗口和压缩

每个模型都有一个**上下文窗口**（它可以看到的 max tokens）。长期运行的聊天累积消息和工具结果；一旦窗口紧张，Clawdbot **压缩**较旧的历史以保持在限制内。

## 什么是压缩
压缩**将较旧的对话总结**为紧凑的摘要条目，并保持最近的消息完整。摘要存储在会话历史中，因此未来的请求使用：
- 压缩摘要
- 压缩点之后的最近消息

压缩**持久化**在会话的 JSONL 历史中。

## 配置
有关 `agents.defaults.compaction` 设置，请参见 [压缩配置和模式](/concepts/compaction)。

## 自动压缩（默认开启）
当会话接近或超过模型的上下文窗口时，Clawdbot 触发自动压缩，并可能使用压缩的上下文重试原始请求。

你将看到：
- 详细模式中的 `🧹 Auto-compaction complete`
- `/status` 显示 `🧹 Compactions: <count>`

在压缩之前，Clawdbot 可以运行**静默内存刷新**回合以将持久化笔记存储到磁盘。有关详细信息和配置，请参见 [内存](/concepts/memory)。

## 手动压缩
使用 `/compact`（可选地带有说明）强制压缩传递：
```
/compact Focus on decisions and open questions
```

## 上下文窗口源
上下文窗口是模型特定的。Clawdbot 使用配置的提供者目录中的模型定义来确定限制。

## 压缩 vs 修剪
- **压缩**：总结并**持久化**在 JSONL 中。
- **会话修剪**：仅修剪旧的**工具结果**，**内存中**，每个请求。

有关修剪详细信息，请参见 [/concepts/session-pruning](/concepts/session-pruning)。

## 提示
- 当会话感觉过时或上下文膨胀时，使用 `/compact`。
- 大型工具输出已被截断；修剪可以进一步减少工具结果的累积。
- 如果你需要全新的开始，`/new` 或 `/reset` 启动新的会话 ID。
